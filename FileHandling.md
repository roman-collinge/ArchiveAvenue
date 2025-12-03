# File Handling Code
This markdown file contains code relating to the handling of files.

## 1. Convert docx files to html with inline images
```python
import base64
import io

import mammoth


def docx_bytes_to_html_bytes(docx_bytes: bytes) -> bytes:
    """Convert DOCX → HTML with inline base64-encoded images."""
    def _convert_image(image):
        with image.open() as image_bytes:
            encoded = base64.b64encode(image_bytes.read()).decode("ascii")
        return {
            "src": f"data:{image.content_type};base64,{encoded}"
        }

    with io.BytesIO(docx_bytes) as docx_stream:  # type: ignore[arg-type]
        result = mammoth.convert_to_html(
            docx_stream,
            convert_image=mammoth.images.img_element(_convert_image),
        )
        html = result.value

    return html.encode("utf-8")
```