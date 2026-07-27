=== SEMANTIC HIERARCHICAL TREE PREVIEW ===

└── 📂 [SECTION HEAD] 1. Executive Summary
        ├── 📝 [PARAGRAPH] This document outlines the quarterly financial performance...
        ├── 📝 [PARAGRAPH] We observed unprecedented growth vectors within our cloud...

└── 📂 [SECTION HEAD] 2. Regional Breakdown
        ├── 📝 [PARAGRAPH] Below is the compiled performance data by territory:
        ├── 📊 [TABLE] (Grid size: 5 rows)
        ├── 📂 [SECTION HEAD] 2.1 North American Logistics
                ├── • [LIST_ITEM] Scaled active logistics nodes by roughly 14% YoY.
                ├── • [LIST_ITEM] Cleared operational bottlenecks in the eastern sector.











---------------------------------------------

import json
from docling.datamodel.document import TextItem, TableItem

def build_semantic_hierarchy(doc_object):
    """
    Post-processes a Docling document object stream to construct a strict
    parent-child semantic tree based on heading hierarchy.
    """
    # 1. Initialize the root container of our custom tree
    tree_root = {
        "node_id": "root",
        "type": "root",
        "text": "Document Root",
        "children": []
    }
    
    # Track the active path of parent headers. 
    # Index matches the heading level (e.g., active_parents[1] is the current H1)
    active_parents = {0: tree_root}
    current_level = 0

    # 2. Iterate through the sequential elements extracted by Docling
    for element in doc_object.elements:
        
        # --- CASE A: ELEMENT IS A HEADING ---
        if isinstance(element, TextItem) and element.label == "heading":
            # Determine the heading level. Fallback to extracting Markdown symbols if level missing
            level = getattr(element, "level", None)
            if level is None:
                # Fallback: Count '#' if parsed as markdown text string
                stripped = element.text.strip()
                level = len(stripped) - len(stripped.lstrip('#')) if stripped.startswith('#') else 1
            
            # Bound level tracking safety limit (1-based index)
            level = max(1, level)
            
            # Format the heading node
            heading_node = {
                "node_id": getattr(element, "self_ref", f"heading_{id(element)}"),
                "type": "heading",
                "level": level,
                "text": element.text.strip(),
                "children": []
            }
            
            # Find the logical parent header. 
            # Walk backwards until we find a header with a lower level number than the current one.
            lookup_level = level - 1
            while lookup_level > 0 and lookup_level not in active_parents:
                lookup_level -= 1
                
            parent_node = active_parents[lookup_level]
            parent_node["children"].append(heading_node)
            
            # Update our active tracking pointer map
            active_parents[level] = heading_node
            current_level = level
            
            # Flush out deeper, stale heading pointers from previous sections
            stale_levels = [l for l in active_parents.keys() if l > level]
            for sl in stale_levels:
                del active_parents[sl]

        # --- CASE B: ELEMENT IS A TABLE ---
        elif isinstance(element, TableItem):
            table_node = {
                "node_id": getattr(element, "self_ref", f"table_{id(element)}"),
                "type": "table",
                "text": "[Table Data Block]",
                "matrix": element.data.model_dump() if hasattr(element.data, "model_dump") else None,
                "children": []
            }
            # Append directly to the lowest active section header
            active_parents[current_level]["children"].append(table_node)

        # --- CASE C: ELEMENT IS CONTENT (Paragraph, List Item, etc.) ---
        elif isinstance(element, TextItem):
            content_node = {
                "node_id": getattr(element, "self_ref", f"content_{id(element)}"),
                "type": element.label,  # "paragraph", "list_item", "caption"
                "text": element.text.strip(),
                "children": []
            }
            # Append directly to the lowest active section header
            active_parents[current_level]["children"].append(content_node)

    return tree_root

# ==========================================
# EXECUTION & EXPORT EXAMPLE
# ==========================================
# Run conversion using your previously configured converter
result = converter.convert("poorly_structured.pdf")

# Generate the hierarchical JSON tree map
hierarchical_tree = build_semantic_hierarchy(result.document)

# Export the structured relationships file to disk
with open("document_hierarchy.json", "w", encoding="utf-8") as f:
    json.dump(hierarchical_tree, f, indent=2, ensure_ascii=False)

print("Successfully exported strict hierarchical JSON relationship tree!")



=============================================


{
  "node_id": "root",
  "type": "root",
  "text": "Document Root",
  "children": [
    {
      "node_id": "#/elements/0",
      "type": "heading",
      "level": 1,
      "text": "1. Introduction",
      "children": [
        {
          "node_id": "#/elements/1",
          "type": "paragraph",
          "text": "This application handles nested workflows safely.",
          "children": []
        },
        {
          "node_id": "#/elements/2",
          "type": "heading",
          "level": 2,
          "text": "1.1 System Architecture",
          "children": [
            {
              "node_id": "#/elements/3",
              "type": "list_item",
              "text": "Component A: Input parsing layer.",
              "children": []
            }
          ]
        }
      ]
    }
  ]
}


==============================================

To transform this nested JSON tree into hierarchical chunks for a Retrieval-Augmented Generation (RAG) system, you must flatten the tree into standalone Document or Node objects. Each node must explicitly contain its own text plus its entire parent lineage metadata.This allows you to implement Small-to-Large Retrieval (retrieving a specific bullet point but feeding the entire parent section to the LLM) or Hierarchical RAG (searching by section header first, then drilling down).


Implementation with LlamaIndexLlamaIndex uses a native TextNode object. By linking parent and child nodes using NodeRelationship.PARENT and NodeRelationship.CHILD, LlamaIndex's RecursiveRetriever can automatically fetch parent context when a child matches.

from llamaindex.core.schema import TextNode, NodeRelationship

def flatten_tree_to_llamaindex_nodes(tree_node, parent_node_id=None, lineage_headers=[]):
    """
    Recursively flattens the custom JSON hierarchy tree into LlamaIndex TextNodes
    while maintaining parent-child lineage and breadcrumbs.
    """
    nodes = []
    current_lineage = list(lineage_headers)
    
    # Track section headers to create semantic breadcrumbs for children
    if tree_node["type"] == "heading":
        current_lineage.append(tree_node["text"])
        
    # Construct a TextNode for content blocks (paragraphs, lists, tables) or headings
    if tree_node["type"] != "root" and tree_node["text"]:
        # Build clean metadata payload for context injection
        metadata = {
            "section_lineage": " > ".join(current_lineage),
            "type": tree_node["type"]
        }
        if "level" in tree_node:
            metadata["heading_level"] = tree_node["level"]

        # Instantiate LlamaIndex text node
        node = TextNode(
            text=tree_node["text"],
            id_=tree_node["node_id"],
            metadata=metadata,
            excluded_llm_metadata_keys=["type", "heading_level"], # Don't confuse the LLM
        )
        
        # Explicitly map the structural parent relationship
        if parent_node_id:
            node.relationships[NodeRelationship.PARENT] = TextNode(id_=parent_node_id)
            
        nodes.append(node)
        current_parent_id = tree_node["node_id"]
    else:
        current_parent_id = parent_node_id

    # Recurse down into children and map the inverse child relationships
    for child in tree_node.get("children", []):
        child_nodes = flatten_tree_to_llamaindex_nodes(
            child, 
            parent_node_id=current_parent_id, 
            lineage_headers=current_lineage
        )
        
        # Link children back to the active parent node if it exists
        if nodes and child_nodes:
            nodes[0].relationships[NodeRelationship.CHILD] = child_nodes[0]
            
        nodes.extend(child_nodes)
        
    return nodes

# --- HOW TO RUN WITH LLAMAINDEX ---
# 1. Transform your previous hierarchical_tree output
llamaindex_nodes = flatten_tree_to_llamaindex_nodes(hierarchical_tree)

# 2. Build index as usual
# from llamaindex.core import VectorStoreIndex
# index = VectorStoreIndex(llamaindex_nodes)


Implementation with LangChainLangChain uses a standard Document object. To handle hierarchies, LangChain relies on the ParentDocumentRetriever pattern, where you store small, granular sub-chunks in a vector database but map their IDs to a full parent text block stored in an in-memory document store.

from langchain_core.documents import Document

def flatten_tree_to_langchain_documents(tree_node, parent_id=None, lineage_headers=[]):
    """
    Flattens the custom JSON tree into LangChain Document instances containing
    metadata pointers for ParentDocumentRetriever tracking.
    """
    docs = []
    current_lineage = list(lineage_headers)
    
    if tree_node["type"] == "heading":
        current_lineage.append(tree_node["text"])
        
    if tree_node["type"] != "root" and tree_node["text"]:
        # Metadata configuration tracking parent connections
        metadata = {
            "source_id": tree_node["node_id"],
            "parent_id": parent_id if parent_id else "root",
            "section_path": " / ".join(current_lineage),
            "element_type": tree_node["type"]
        }
        
        doc = Document(
            page_content=tree_node["text"],
            metadata=metadata
        )
        docs.append(doc)
        next_parent_id = tree_node["node_id"]
    else:
        next_parent_id = parent_id

    # Recurse through branches
    for child in tree_node.get("children", []):
        docs.extend(
            flatten_tree_to_langchain_documents(
                child, 
                parent_id=next_parent_id, 
                lineage_headers=current_lineage
            )
        )
        
    return docs

# --- HOW TO RUN WITH LANGCHAIN ---
# 1. Transform your previous hierarchical_tree output
langchain_docs = flatten_tree_to_langchain_documents(hierarchical_tree)

# 2. Load into your vector store
# vectorstore.add_documents(langchain_docs)


Why this changes your RAG application's performancePerfect Contextual Awareness: When a child paragraph or single bullet point is retrieved, it carries its section_lineage metadata (e.g., 2. Regional Breakdown > 2.1 North American Logistics). The LLM immediately understands the scope of the paragraph without needing the surrounding text.Deterministic Layout Reconstruction: It eliminates the classic character-count chunking failure where an overlapping text window breaks a sentence or splits a single table across two database vectors.Would you like to explore:How to configure LlamaIndex's RecursiveRetriever to automatically swap child text with full parent text during search execution?How to construct custom Prompt Templates that effectively inject this section_lineage metadata into your generation layer?

