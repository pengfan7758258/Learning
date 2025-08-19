# scrape_webpages
```python
from langchain_core.tools import tool
from langchain_community.document_loaders import WebBaseLoader

@tool
def scrape_webpages(urls: List[str]) -> str:
	"""Use requests and bs4 to scrape the provided web pages for detailed information."""
	loader = WebBaseLoader(urls)
	docs = loader.load()
	return "\n\n".join(
		[
			f'<Document name="{doc.metadata.get("title", "")}">\n{doc.page_content}\n</Document>'
			for doc in docs
		]
	)
```
- 具体爬虫细节可以使用requests or scrapy重构：请求头，反爬等等