# 5.5 问答接口开发（/chat）

## 流式 HTTP 响应：SSE

普通的 HTTP 响应是「一次性」的：客户端发请求，服务器把结果全部准备好，一次性返回。

**SSE（Server-Sent Events）** 是一种特殊的 HTTP 响应模式：服务器可以持续向客户端推送数据，直到结束。这正是聊天打字机效果的技术基础。

```
客户端                       服务器
  │                             │
  ├──── POST /api/chat ──────►  │
  │                             │ 检索知识库...
  │ ◄─── "根据" ────────────── │ 生成第1个字
  │ ◄─── "您" ─────────────── │ 生成第2个字
  │ ◄─── "上传" ───────────── │ 生成第3个字
  │          ... （流式输出）   │
  │ ◄─── [data: [DONE]] ──── │ 完成
```

---

## 后端：流式接口实现

```python
# app/api/chat.py
import json
import uuid
from fastapi import APIRouter, Depends, HTTPException
from fastapi.responses import StreamingResponse
from sqlalchemy.orm import Session
from pydantic import BaseModel
from app.core.database import get_db
from app.models.models import User, QARecord
from app.services.auth import get_current_user
from app.services.rag import generate_stream

router = APIRouter()

class ChatRequest(BaseModel):
    question: str
    session_id: str = None  # 可选，用于关联同一会话的多条记录

@router.post("/chat")
def chat(
    req: ChatRequest,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """问答接口（SSE 流式响应）"""
    if not req.question.strip():
        raise HTTPException(status_code=400, detail="问题不能为空")
    
    session_id = req.session_id or str(uuid.uuid4())
    
    def event_generator():
        """SSE 事件生成器"""
        full_answer = ""
        source_chunks = []
        
        try:
            gen = generate_stream(req.question)
            while True:
                try:
                    chunk = next(gen)
                    full_answer += chunk
                    # SSE 格式：data: <内容>\n\n
                    yield f"data: {json.dumps({'type': 'text', 'content': chunk}, ensure_ascii=False)}\n\n"
                except StopIteration as e:
                    # 生成器结束，获取完整结果
                    result = e.value
                    if result:
                        source_chunks = result.get("source_chunks", [])
            
        except Exception as e:
            yield f"data: {json.dumps({'type': 'error', 'content': str(e)})}\n\n"
        
        # 保存问答记录
        try:
            record = QARecord(
                user_id=current_user.id,
                session_id=session_id,
                question=req.question,
                answer=full_answer,
                source_chunks=source_chunks
            )
            db.add(record)
            db.commit()
        except Exception as e:
            pass  # 记录保存失败不影响主流程
        
        # 发送结束信号
        yield f"data: {json.dumps({'type': 'done', 'session_id': session_id})}\n\n"
    
    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no"  # 禁用 Nginx 缓冲（生产环境重要）
        }
    )

@router.get("/chat/history")
def get_history(
    page: int = 1,
    page_size: int = 20,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """获取当前用户的问答历史"""
    query = db.query(QARecord).filter(QARecord.user_id == current_user.id)
    total = query.count()
    records = query.order_by(QARecord.created_at.desc()) \
                   .offset((page-1)*page_size).limit(page_size).all()
    
    return {
        "total": total,
        "items": [{
            "id": r.id,
            "question": r.question,
            "answer": r.answer,
            "created_at": str(r.created_at)
        } for r in records]
    }
```

---

## 本节小结

- SSE 是实现流式打字机效果的标准技术
- `StreamingResponse` + `event_generator()` 是 FastAPI 流式响应的标准模式
- 问答记录在流式输出完成后保存，不影响实时响应
- `data: [内容]\n\n` 是 SSE 事件的固定格式
