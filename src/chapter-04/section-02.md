# 4.2 文档处理：文本切分（Chunking）

## 为什么要切分文档？

直觉上，我们会想把整篇文档向量化，然后检索。但这样做有两个问题：

1. **Embedding 模型有长度限制**：大多数模型最多接受 512~8192 个 Token（大约几百到几千个汉字）
2. **检索精度下降**：如果一个向量代表 1000 字的文档，这个向量是「全文语义的平均」，很难精准匹配用户的具体问题

**切分的目标**：把文档切成大小合适的片段，使每个片段：
- 语义相对独立（一个完整的意思）
- 长度在模型可处理范围内（300~800 字为宜）

---

## 固定长度切分 vs 语义切分

### 固定长度切分（Simple Chunking）
```
全文: ABCDEFGHIJ...
切分: [ABCD] [EFGH] [IJKL] ...
```

优点：简单、可预测
缺点：可能在句子中间切断，破坏语义

### 带重叠的固定长度切分（Overlapping Chunks）
```
全文: ABCDEFGHIJ...
切分: [ABCDE] [CDEFG] [EFGHI] ...（重叠 CD、EF）
```

重叠（Overlap）的作用：防止关键信息恰好在两段的边界处被切断。

### 语义切分（Semantic Chunking）
按段落、章节等自然边界切分。

---

## 实战：用 LangChain 切分文档

安装依赖：
```bash
pip install langchain langchain-community pypdf python-docx
```

### 文档处理服务 `app/services/doc_processor.py`

```python
import os
import logging
from pathlib import Path
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.document_loaders import (
    PyPDFLoader, TextLoader, Docx2txtLoader
)

logger = logging.getLogger(__name__)

# 按文件类型选择加载器
LOADERS = {
    ".pdf":  PyPDFLoader,
    ".txt":  TextLoader,
    ".docx": Docx2txtLoader,
}

def load_document(file_path: str) -> str:
    """加载文档，返回纯文本"""
    ext = Path(file_path).suffix.lower()
    loader_cls = LOADERS.get(ext)
    if not loader_cls:
        raise ValueError(f"不支持的文件类型: {ext}")
    
    loader = loader_cls(file_path)
    docs = loader.load()
    # 合并所有页面的文本
    return "\n\n".join(doc.page_content for doc in docs)


def split_text(text: str, chunk_size: int = 500, chunk_overlap: int = 50) -> list[str]:
    """
    文本切分
    
    Args:
        text: 待切分的全文
        chunk_size: 每个片段的目标字符数（默认500）
        chunk_overlap: 相邻片段的重叠字符数（默认50）
    
    Returns:
        切分后的文本片段列表
    """
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=chunk_overlap,
        # 优先在这些分隔符处切分，保持语义完整
        separators=["\n\n", "\n", "。", "！", "？", "；", "，", " ", ""],
        length_function=len,  # 按字符数计算长度
    )
    
    chunks = splitter.split_text(text)
    logger.info(f"文本切分完成：{len(text)} 字 → {len(chunks)} 个片段")
    return chunks
```

---

## 切分参数的选择

| 场景 | chunk_size | chunk_overlap | 理由 |
|------|-----------|---------------|------|
| 通用文档 | 500 | 50 | 平衡精度和覆盖 |
| 技术文档（代码较多） | 800 | 100 | 代码片段通常较长 |
| 聊天记录 | 200 | 20 | 每条消息本身就短 |
| 学术论文 | 600 | 80 | 段落通常较长 |

---

## 本节小结

- 切分文档是 RAG 管道的关键步骤，直接影响检索质量
- `RecursiveCharacterTextSplitter` 优先在语义边界（段落、句子）处切分
- `chunk_overlap` 防止关键信息丢失在片段边界
- 默认 500 字 / 50 字重叠是个不错的起点，根据文档类型调整
