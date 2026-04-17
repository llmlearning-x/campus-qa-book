# 4.1 知识库模块 CRUD

## 文档上传处理完整 Pipeline

```python
# app/api/knowledge.py
import os
import uuid
from fastapi import APIRouter, Depends, HTTPException, UploadFile, File, BackgroundTasks
from sqlalchemy.orm import Session
from app.core.database import get_db
from app.core.config import settings
from app.models.models import Document, DocumentChunk, User
from app.services.auth import require_admin, get_current_user
from app.services.doc_processor import load_document, split_text
from app.services.embedder import embed_batch
from app.services.vector_store import vector_store
import logging

router = APIRouter()
logger = logging.getLogger(__name__)

@router.post("/documents")
async def upload_document(
    file: UploadFile = File(...),
    background_tasks: BackgroundTasks = BackgroundTasks(),
    db: Session = Depends(get_db),
    current_user: User = Depends(require_admin)
):
    """上传文档并异步处理"""
    # 校验文件类型
    allowed_types = {".pdf", ".txt", ".docx"}
    ext = os.path.splitext(file.filename)[1].lower()
    if ext not in allowed_types:
        raise HTTPException(status_code=400, detail=f"不支持的文件类型，支持：{allowed_types}")
    
    # 校验文件大小
    content = await file.read()
    if len(content) > settings.MAX_FILE_SIZE:
        raise HTTPException(status_code=400, detail="文件大小超过限制（50MB）")
    
    # 保存文件到磁盘
    os.makedirs(settings.UPLOAD_DIR, exist_ok=True)
    filename = f"{uuid.uuid4()}{ext}"
    file_path = os.path.join(settings.UPLOAD_DIR, filename)
    with open(file_path, "wb") as f:
        f.write(content)
    
    # 创建数据库记录
    doc = Document(
        filename=file.filename,
        file_path=file_path,
        file_size=len(content),
        file_type=ext.lstrip("."),
        status="pending",
        uploaded_by=current_user.id
    )
    db.add(doc)
    db.commit()
    db.refresh(doc)
    
    # 后台异步处理文档（不阻塞 HTTP 响应）
    background_tasks.add_task(process_document, doc.id, file_path)
    
    return {
        "doc_id": doc.id,
        "filename": file.filename,
        "status": "pending",
        "message": "文件上传成功，正在后台处理..."
    }


def process_document(doc_id: int, file_path: str):
    """后台任务：切分 + Embedding + 存入向量库"""
    from app.core.database import SessionLocal
    db = SessionLocal()
    
    try:
        doc = db.query(Document).filter(Document.id == doc_id).first()
        doc.status = "processing"
        db.commit()
        
        # Step 1: 提取文本
        logger.info(f"[Doc {doc_id}] 开始提取文本...")
        text = load_document(file_path)
        
        # Step 2: 文本切分
        logger.info(f"[Doc {doc_id}] 开始切分文本...")
        chunks = split_text(text, chunk_size=500, chunk_overlap=50)
        
        # Step 3: 向量化
        logger.info(f"[Doc {doc_id}] 开始 Embedding（{len(chunks)} 个片段）...")
        vectors = embed_batch(chunks)
        
        # Step 4: 存入数据库和向量库
        chunk_objects = []
        for i, (chunk_text, vector) in enumerate(zip(chunks, vectors)):
            chunk = DocumentChunk(
                doc_id=doc_id,
                chunk_index=i,
                content=chunk_text
            )
            db.add(chunk)
            db.flush()  # 获取 chunk.id
            chunk_objects.append((chunk, vector))
        
        # 批量添加到 FAISS
        chunk_ids = [c.id for c, _ in chunk_objects]
        contents  = [c.content for c, _ in chunk_objects]
        vecs      = [v for _, v in chunk_objects]
        vector_ids = vector_store.add_vectors(vecs, chunk_ids, contents)
        
        # 更新 vector_id
        for (chunk, _), vid in zip(chunk_objects, vector_ids):
            chunk.vector_id = vid
        
        doc.chunk_count = len(chunks)
        doc.status = "done"
        db.commit()
        logger.info(f"[Doc {doc_id}] 处理完成！共 {len(chunks)} 个片段")
        
    except Exception as e:
        logger.error(f"[Doc {doc_id}] 处理失败：{e}", exc_info=True)
        doc = db.query(Document).filter(Document.id == doc_id).first()
        if doc:
            doc.status = "failed"
            db.commit()
    finally:
        db.close()


@router.get("/documents")
def list_documents(
    page: int = 1, page_size: int = 10,
    db: Session = Depends(get_db),
    _: User = Depends(get_current_user)
):
    """获取文档列表"""
    query = db.query(Document)
    total = query.count()
    docs = query.order_by(Document.created_at.desc()) \
                .offset((page-1)*page_size).limit(page_size).all()
    return {
        "total": total,
        "items": [{
            "id": d.id, "filename": d.filename, "status": d.status,
            "chunk_count": d.chunk_count, "file_size": d.file_size,
            "created_at": str(d.created_at)
        } for d in docs]
    }


@router.delete("/documents/{doc_id}")
def delete_document(
    doc_id: int,
    db: Session = Depends(get_db),
    _: User = Depends(require_admin)
):
    """删除文档（同时清理向量库）"""
    doc = db.query(Document).filter(Document.id == doc_id).first()
    if not doc:
        raise HTTPException(status_code=404, detail="文档不存在")
    
    # 获取所有 chunk_id 用于清理向量库
    chunk_ids = {c.id for c in doc.chunks}
    
    # 从 FAISS 删除向量
    if chunk_ids:
        vector_store.remove_by_chunk_ids(chunk_ids)
    
    # 删除磁盘文件
    if os.path.exists(doc.file_path):
        os.remove(doc.file_path)
    
    # 删除数据库记录（chunks 级联删除）
    db.delete(doc)
    db.commit()
    
    return {"message": f"文档「{doc.filename}」已删除"}
```

---

## 本节小结

- `BackgroundTasks` 实现异步文档处理，上传后立即返回，不阻塞用户
- 处理 Pipeline：提取文本 → 切分 → Embedding → 存 FAISS + MySQL
- 删除文档时要同时清理向量库，保持数据一致性
- 文件用 UUID 重命名，避免同名文件冲突
