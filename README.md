import os
from docling.datamodel.base_models import InputFormat
from docling.datamodel.pipeline_options import (
    PdfPipelineOptions, 
    TableStructureOptions,
    TableFormerMode
)
from docling.document_converter import DocumentConverter, PdfFormatOption
from docling.datamodel.document import LayoutItem, TableItem, TextItem

# ==========================================
# 1. OPTIMIZED PIPELINE CONFIGURATION
# ==========================================

pipeline_options = PdfPipelineOptions()

# Enforce visual layout structure parsing over broken internal tags
pipeline_options.do_table_structure = True 

# Use high-accuracy models to handle messy or borderless grids
pipeline_options.table_structure_options = TableStructureOptions(
    mode=TableFormerMode.ACCURATE
)

# Tell Docling to trust visual boundaries instead of corrupted PDF layout streams
pipeline_options.table_structure_options.do_cell_matching = False

# Enable visual OCR fallback to correct missing text blocks and corrupted font maps
pipeline_options.do_ocr = True
pipeline_options.ocr_options.lang = ["en"]  # Adjust language codes as necessary

# Bind options to the PDF reader format
format_options = {
    InputFormat.PDF: PdfFormatOption(pipeline_options=pipeline_options)
}

# Instantiate the engine
converter = DocumentConverter(format_options=format_options)


# ==========================================
# 2. RUN CONVERSION
# ==========================================

# Replace with the path to your poorly-structured PDF file
pdf_path = "poorly_structured.pdf" 
result = converter.convert(pdf_path)
doc_object = result.document  # This contains the rich programmatic object tree


# ==========================================
# 3. CUSTOM HIERARCHICAL TREE VISUALIZATION
# ==========================================

def display_document_tree(node, depth=0):
    """
    Recursively steps down through the structural nodes of the Docling Document object 
    to visually represent parent-child sectional relationships.
    """
    indent = "    " * depth
    
    # 1. Handle Structural Section Headings
    if isinstance(node, TextItem) and node.label == "heading":
        # Draw section anchors to visibly separate structural shifts
        print(f"\n{indent}└── 📂 [SECTION HEAD] {node.text.strip()}")
        next_depth = depth + 1
        
    # 2. Handle Tables Embedded In Sections
    elif isinstance(node, TableItem):
        # Indicate a table child is bound strictly under the active section
        row_count = len(node.data.rows) if node.data else 0
        print(f"{indent}    ├── 📊 [TABLE] (Grid size: {row_count} rows)")
        next_depth = depth
        
    # 3. Handle Paragraphs / Lists / Caption Items
    elif isinstance(node, TextItem):
        # Shorten text snippets in the tree preview to keep it scannable
        snippet = node.text.strip().replace('\n', ' ')
        preview = snippet if len(snippet) <= 60 else f"{snippet[:57]}..."
        
        # Differentiate bullet lists from pure body blocks visually
        icon = "•" if node.label == "list_item" else "📝"
        print(f"{indent}    ├── {icon} [{node.label.upper()}] {preview}")
        next_depth = depth
        
    # 4. Catch-all for generic layout structural items
    else:
        next_depth = depth

    # Recursively cycle through every immediate structural child item
    # Docling nodes maintain strict, chronological parent-child lists internally
    if hasattr(node, "children") and node.children:
        for child in node.children:
            display_document_tree(child, depth=next_depth)

# Execute the tree rendering script starting at the root body layer
print("=== SEMANTIC HIERARCHICAL TREE PREVIEW ===")
if hasattr(doc_object, "body") and doc_object.body:
    display_document_tree(doc_object.body)
else:
    # Fallback pattern if structural layout mapping yields a flat stream
    for element in doc_object.elements:
        display_document_tree(element)
