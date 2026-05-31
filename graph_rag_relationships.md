how to apply graph rag with langraph langchain where the documents are all policy related?

Applying Graph RAG to policy documents (like HR handbooks, compliance guidelines, or legal terms) is a highly effective architecture. Standard vector RAG often fails here because policies are highly interconnected: a policy on "Remote Work" might depend heavily on the "Equipment Provision" policy and "Security Protocols."

By combining a Graph Database (like Neo4j), LangChain (for data ingestion and LLM utilities), and LangGraph (to build a stateful, agentic retrieval workflow), you can build a system capable of multi-hop reasoning over complex guidelines.  
Towards AI

1. Defining the Policy Graph Ontology
Before writing code, define how your policies connect. A standard vector store treats paragraphs as isolated islands. A Knowledge Graph links them systematically. For policies, use this ontology:

Nodes: * Document (e.g., Employee Handbook 2026)

Section / Policy (e.g., Section 4.2: Parental Leave)

Entity (e.g., Full-Time Employee, HR Department, Maternity)

Relationships: * (:Document)-[:HAS_SECTION]->(:Section)

(:Section)-[:SUPERSEDES]->(:Section) (Crucial for version control!)

(:Section)-[:REQUIRES_APPROVAL_FROM]->(:Entity)

(:Section)-[:REFERENCES]->(:Section) (Cross-references)

2. Ingestion & Graph Construction (LangChain)
First, use LangChain's LLMGraphTransformer alongside a graph database wrapper like Neo4jGraph to parse documents into entities and relationships, while saving vector embeddings on the text chunks for Hybrid Search.

Python
import os
from langchain_community.graphs import Neo4jGraph
from langchain_experimental.graph_transformers import LLMGraphTransformer
from langchain_openai import ChatOpenAI
from langchain_core.documents import Document

# 1. Connect to Neo4j
graph = Neo4jGraph(
    url=os.environ["NEO4J_URI"], 
    username=os.environ["NEO4J_USERNAME"], 
    password=os.environ["NEO4J_PASSWORD"]
)

# 2. Define your Policy-Specific Entities and Relationships
llm = ChatOpenAI(model="gpt-4o", temperature=0)
transformer = LLMGraphTransformer(
    llm=llm,
    allowed_nodes=["Policy", "Section", "Role", "Requirement", "Action"],
    allowed_relationships=["REFERENCES", "SUPERSEDES", "APPLIES_TO", "REQUIRES_APPROVAL"]
)

# 3. Process documents (Assuming you chunked your PDF/Word policy files)
docs = [
    Document(page_content="Section 4.1: Maternity Leave applies to Full-Time Employees. It requires approval from the HR Department and references the FMLA guidelines.")
]

graph_docs = transformer.convert_to_graph_documents(docs)

# 4. Write to Graph DB
graph.add_graph_documents(graph_docs, baseEntityLabel=True, include_source=True)
💡 Pro-Tip for Policies: Also build a standard vector index directly inside Neo4j on the Document or Section text properties. This allows you to perform Vector Search to find the entry point, and Cypher Queries to pull adjacent cross-referenced rules.

3. Designing the Agent Workflow (LangGraph)
Policy questions are rarely straightforward. A user might ask: "What is my allowance for parental leave, and do I need my manager's approval?" Instead of a linear chain, use LangGraph to build a stateful agent loop that:

Extracts the core intent and entities.

Performs a hybrid vector + graph lookup.

Checks if the retrieved policies reference other sections (multi-hop).

Loops back to fetch cross-referenced policies if needed.

Formulates the final compliant answer.

Step-by-Step Graph Architecture
Python
from typing import Annotated, List, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langchain_core.messages import BaseMessage

# Define the Agent State
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]
    retrieved_policies: List[str]
    cross_references: List[str]
    loop_count: int

# --- NODE 1: Query Analyzer / Router ---
def analyze_query(state: AgentState):
    # LLM determines the core entity (e.g., "Maternity Leave")
    # and generates optimized keywords/Cypher fragments.
    return {"loop_count": 0}

# --- NODE 2: Hybrid Graph Retriever ---
def retrieve_knowledge(state: AgentState):
    user_query = state["messages"][-1].content
    
    # 1. Vector Search: Find closest policy text
    # 2. Graph Search: Query Neo4j for related nodes 
    # Example Cypher pattern: MATCH (p:Policy)-[:REFERENCES|REQUIRES_APPROVAL]->(target)
    
    # Simulating found context and cross-references found in metadata/edges
    found_context = "Section 4.1: Maternity Leave requires HR Approval."
    found_refs = ["Section 5.0 (HR Approval Process)"] 
    
    return {
        "retrieved_policies": [found_context],
        "cross_references": found_refs,
        "loop_count": state["loop_count"] + 1
    }

# --- CONDITIONAL EDGE: Decide if we need more context ---
def should_fetch_references(state: AgentState):
    # If we found cross-referenced rules, and haven't looped too much, go get them
    if state["cross_references"] and state["loop_count"] < 2:
        return "fetch_hops"
    return "generate_answer"

# --- NODE 3: Multi-Hop Fetcher ---
def fetch_hops(state: AgentState):
    # Query Neo4j explicitly for the IDs/Names in state["cross_references"]
    additional_context = "Section 5.0: HR Approval takes 5 business days."
    
    return {
        "retrieved_policies": state["retrieved_policies"] + [additional_context],
        "cross_references": [] # Clear out so we don't loop forever
    }

# --- NODE 4: Generator ---
def generate_answer(state: AgentState):
    context = "\n".join(state["retrieved_policies"])
    prompt = f"Answer based strictly on this policy context:\n{context}"
    # Call LLM here...
    return {"messages": [("assistant", "Parsed response...")]}
Compiling the Flow
Now, assemble these components using LangGraph's StateGraph:

Python
workflow = StateGraph(AgentState)

# Add Nodes
workflow.add_node("analyze_query", analyze_query)
workflow.add_node("retrieve_knowledge", retrieve_knowledge)
workflow.add_node("fetch_hops", fetch_hops)
workflow.add_node("generate_answer", generate_answer)

# Add Edges
workflow.add_edge(START, "analyze_query")
workflow.add_edge("analyze_query", "retrieve_knowledge")

# Conditional Routing based on whether cross-referenced policies exist
workflow.add_conditional_edges(
    "retrieve_knowledge",
    should_fetch_references,
    {
        "fetch_hops": "fetch_hops",
        "generate_answer": "generate_answer"
    }
)
workflow.add_edge("fetch_hops", "generate_answer")
workflow.add_edge("generate_answer", END)

# Compile Agent
policy_rag_agent = workflow.compile()
Why this Pattern excels for Policies
Strict Factuality: By wrapping this in LangGraph, you can introduce a "Guardrail Node" right before generate_answer that verifies if the generated text contradicts any extracted graph relationships (e.g., if the graph says Requires Approval but the LLM answer says 'No approval needed').

Handling Amending Documents: Policies change. If "Policy Memo B" has a [:SUPERSEDES] relationship to "Policy A", your Cypher queries can explicitly filter out out-of-date text chunks during the retrieval step, preventing the LLM from serving stale information.

how to extract the relationship between paragraphs and documents?

Extracting relationships between structural layers (like Documents, Chapters, Sections, and Paragraphs) is one of the highest-value steps for policy documents. Unlike extracting general entities (like names or dates), document hierarchy is mostly sequential and structural.

Because of this, relying purely on an LLM to "guess" these relationships is slow and expensive. The best practice is a Hybrid Ingestion Pipeline: use deterministic Python parsing for structural relationships, and use LLMs for semantic semantic relationships.

Here is how to build this extraction pipeline using LangChain and Python.

1. The Target Graph Schema
Before extracting, visualize what you are trying to build in your graph database:

[Document] ──(:HAS_SECTION)──> [Section/Header] ──(:HAS_PARAGRAPH)──> [Paragraph/Chunk]
                                                                            │
                                                                       (:REFERENCES)
                                                                            ▼
                                                                     [Another Section]
2. Structural Extraction (Deterministic Python)
Policy documents are heavily structured. If your document is a Markdown file, HTML, or structured PDF, you can use LangChain’s text splitters to extract the layout hierarchy automatically.

Here is how to extract Document -> Section -> Paragraph parent-child relationships using LangChain's MarkdownHeaderTextSplitter:

Python
from langchain_text_splitters import MarkdownHeaderTextSplitter
import uuid

# 1. Sample Markdown representation of a policy
policy_document = """
# HR Leave Policy 2026
## 1. Parental Leave
This section outlines parental leave. Full-time employees get 12 weeks.
## 2. Remote Work
Employees can work remotely up to 3 days per week. This adheres to Section 1.
"""

# 2. Define headers to split on
headers_to_split_on = [
    ("#", "Document_Title"),
    ("##", "Section_Title"),
]

markdown_splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers_to_split_on)
chunks = markdown_splitter.split_text(policy_document)

# 3. Transform into structured graph records
graph_nodes = []
graph_edges = []

doc_id = "doc_hr_leave_2026"  # Base document ID

for chunk in chunks:
    section_title = chunk.metadata.get("Section_Title", "General")
    doc_title = chunk.metadata.get("Document_Title", "Untitled")
    paragraph_text = chunk.page_content
    paragraph_id = str(uuid.uuid4())[:8]
    
    # Track the Paragraph Node
    graph_nodes.append({
        "id": paragraph_id,
        "type": "Paragraph",
        "text": paragraph_text
    })
    
    # Track Section Node (using text as ID or creating a slug)
    section_id = f"sec_{section_title.lower().replace(' ', '_')}"
    graph_nodes.append({
        "id": section_id,
        "type": "Section",
        "title": section_title
    })
    
    # Establish structural edges deterministically
    graph_edges.append({"source": doc_id, "target": section_id, "type": "HAS_SECTION"})
    graph_edges.append({"source": section_id, "target": paragraph_id, "type": "HAS_PARAGRAPH"})
3. Semantic Extraction (LLM-Based)
Now that you have your paragraphs tied securely to their parent documents and sections, you need to extract the semantic lines between paragraphs—such as cross-references (e.g., "This adheres to Section 1").

This is where you pass individual paragraph chunks to an LLM using LangChain Structured Outputs to find internal citations.

Python
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI

# 1. Define the schema for references found inside a paragraph
class PolicyReference(BaseModel):
    target_section: str = Field(description="The name or number of the section being cross-referenced.")
    relationship_type: str = Field(description="Must be either 'REFERENCES', 'SUPERSEDES', or 'MODIFIES'.")

class ParagraphAnalysis(BaseModel):
    references: list[PolicyReference] = Field(description="List of all policy cross-references found in the text.")

# 2. Initialize LLM with structured output
llm = ChatOpenAI(model="gpt-4o", temperature=0)
structured_llm = llm.with_structured_output(ParagraphAnalysis)

# 3. Analyze a paragraph that mentions another section
sample_paragraph = "Employees can work remotely up to 3 days per week. This adheres to Section 1 (Parental Leave)."

analysis = structured_llm.invoke(f"Analyze this policy paragraph for internal cross-references:\n\n{sample_paragraph}")

print(analysis.model_dump_json(indent=2))
LLM Output:
JSON
{
  "references": [
    {
      "target_section": "1. Parental Leave",
      "relationship_type": "REFERENCES"
    }
  ]
)
4. Merging and Loading to Neo4j
Once you have both structural items (from Python) and semantic linkages (from the LLM), you merge them into a unified Cypher query execution block to populate your Graph Database:

Python
from langchain_community.graphs import Neo4jGraph

graph = Neo4jGraph()

# 1. Create Document and Section structure
for edge in graph_edges:
    # Example query to knit the hierarchy together
    # logic depends on if source/target is Document, Section, or Paragraph
    pass

# 2. Link the extracted LLM Cross-References
for ref in analysis.references:
    # Find the current paragraph, find the target section it mentions, and draw the line
    cypher_query = """
    MATCH (p:Paragraph {text: $current_paragraph_text})
    MATCH (target:Section) WHERE target.title CONTAINS $target_title
    MERGE (p)-[:REFERENCES]->(target)
    """
    graph.query(cypher_query, {
        "current_paragraph_text": sample_paragraph,
        "target_title": ref.target_section
    })
Summary of Best Practices
Don't ask LLMs to build the document tree: LLMs can miss paragraph counts or hallucinate section boundaries. Use text splitters (MarkdownHeaderTextSplitter, HTMLHeaderTextSplitter) to dictate structural parents.

Do use LLMs for downstream semantic mapping: Let the LLM read the isolated paragraph to find logic dependencies (SUPERSEDES, REQUIRES_COMPLIANCE_WITH) that simple regex or Python code cannot pick up.

how to encode and store the relationships between paragraphs and documents?

To store and query the relationships between paragraphs and documents effectively, you need a database strategy that accommodates two completely different paradigms:

Graph Connections: Storing the structural and semantic lines (e.g., Document -> Section -> Paragraph and Paragraph -> REFERENCES -> Section).

Vector Embeddings: Encoding the text of the paragraphs into numerical vectors (1536 or 3072 dimensions) so the LLM can find them via semantic search.

Instead of running two separate databases, the industry standard for Graph RAG is to use Neo4j (or an equivalent Graph DB) as a hybrid store. Neo4j natively stores graph nodes/edges and can hold vector embeddings directly on those nodes.

Here is the exact blueprint for encoding your relationships, generating embeddings, and storing them using LangChain and Neo4j.

1. The Strategy: How Data is Represented
You will create three distinct Node labels and map out their relationships:

(:Document {id, title})
     │
  [:HAS_SECTION]
     ▼
(:Section {id, title})
     │
  [:HAS_PARAGRAPH]
     ▼
(:Paragraph {id, text, embedding: [...]}) ──[:REFERENCES]──> (:Section)
2. Ingestion, Embedding, and Storage Script
This complete script takes your structured Python dictionary data (from your parsing step), uses LangChain to generate vector embeddings for the paragraphs, and writes the entire relational web into Neo4j in a single sweep.

Python
import os
import uuid
from langchain_community.graphs import Neo4jGraph
from langchain_openai import OpenAIEmbeddings

# 1. Connect to Neo4j
graph = Neo4jGraph(
    url=os.environ["NEO4J_URI"],
    username=os.environ["NEO4J_USERNAME"],
    password=os.environ["NEO4J_PASSWORD"]
)

# 2. Initialize the Embedding Model
embeddings_model = OpenAIEmbeddings(model="text-embedding-3-large")

# 3. Sample Structured Data Pipeline Output
document_data = {
    "doc_id": "doc_hr_policy_2026",
    "doc_title": "Employee Handbook 2026",
    "sections": [
        {
            "section_id": "sec_parental_leave",
            "section_title": "Section 1: Parental Leave",
            "paragraphs": [
                {
                    "p_id": str(uuid.uuid4())[:8],
                    "text": "Full-time employees receive 12 weeks of paid parental leave. This policy relies heavily on compliance with standard FMLA parameters.",
                    "cross_references": ["Section 3: Family Medical Leave"] # From LLM extraction
                }
            ]
        },
        {
            "section_id": "sec_fmla",
            "section_title": "Section 3: Family Medical Leave",
            "paragraphs": [
                {
                    "p_id": str(uuid.uuid4())[:8],
                    "text": "Family Medical Leave (FMLA) grants protection for up to 12 weeks of unpaid job-protected leave for qualified medical reasons.",
                    "cross_references": []
                }
            ]
        }
    ]
}

# 4. Process and Store inside Neo4j
for section in document_data["sections"]:
    for paragraph in section["paragraphs"]:
        
        # Step A: Compute Vector Embedding for the Paragraph Text
        p_embedding = embeddings_model.embed_query(paragraph["text"])
        
        # Step B: Cypher Query to create the Nodes, structural Edges, and add the vector
        cypher_query = """
        // 1. Merge Document and Section
        MERGE (d:Document {id: $doc_id})
        SET d.title = $doc_title
        
        MERGE (s:Section {id: $section_id})
        SET s.title = $section_title
        
        MERGE (d)-[:HAS_SECTION]->(s)
        
        // 2. Create Paragraph Node with Vector Embedding
        MERGE (p:Paragraph {id: $p_id})
        SET p.text = $p_text,
            p.embedding = $p_embedding
            
        MERGE (s)-[:HAS_PARAGRAPH]->(p)
        """
        
        graph.query(cypher_query, {
            "doc_id": document_data["doc_id"],
            "doc_title": document_data["doc_title"],
            "section_id": section["section_id"],
            "section_title": section["section_title"],
            "p_id": paragraph["p_id"],
            "p_text": paragraph["text"],
            "p_embedding": p_embedding
        })
        
        # Step C: Draw Semantic Cross-Reference Edges
        for ref_title in paragraph["cross_references"]:
            ref_query = """
            MATCH (p:Paragraph {id: $p_id})
            MERGE (target:Section {title: $ref_title})
            MERGE (p)-[:REFERENCES]->(target)
            """
            graph.query(ref_query, {"p_id": paragraph["p_id"], "ref_title": ref_title})

print("Successfully encoded and saved policy graph architecture.")
3. Configuring the Vector Index in Neo4j
Once your nodes, relationships, and raw vector floats are inside the database, you need to tell Neo4j to spin up a vector index on the Paragraph labels. Run this command to initialize it:

Python
# Create Vector Index on Paragraph node embedding properties
graph.query("""
CREATE VECTOR INDEX `paragraph_vector_index` IF NOT EXISTS
FOR (p:Paragraph) ON (p.embedding)
OPTIONS {
  indexConfig: {
    `vector.dimensions`: 3072,
    `vector.similarity_function`: 'cosine'
  }
}
""")
4. How to Query It (The RAG Step)
Now that the connections are stored, your LangGraph retrieval node can execute queries that marry vector math with graph topology.

For example, if a user asks: "Tell me about parental leave requirements," you can do a vector search to find the paragraph, and immediately grab any global sections it references:

Python
# 1. Embed the user's question
query_vector = embeddings_model.embed_query("What are the parental leave parameters?")

# 2. Hybrid Query: Vector Search + Graph Traversal
retrieval_query = """
CALL db.index.vector.queryNodes('paragraph_vector_index', 3, $query_vector) 
YIELD node AS start_paragraph, score

// Traverse up to find the document context
MATCH (doc:Document)-[:HAS_SECTION]->(sec:Section)-[:HAS_PARAGRAPH]->(start_paragraph)

// Travese outward to find any cross-referenced rules
OPTIONAL MATCH (start_paragraph)-[:REFERENCES]->(ref_sec:Section)

RETURN 
    doc.title AS Document,
    sec.title AS CurrentSection,
    start_paragraph.text AS ParagraphText,
    collect(ref_sec.title) AS CrossReferences,
    score
"""

results = graph.query(retrieval_query, {"query_vector": query_vector})
print(results)
The Output Payload for your LLM Prompt:
Instead of just a chunk of unstructured text, your LangGraph node receives a highly contextual payload:

Text Chunks: "Full-time employees receive 12 weeks..."

Hierarchy Context: It came from Employee Handbook 2026 under Section 1.

Downstream Tasks: It lets the LLM know: "Hey, this paragraph explicitly references 'Section 3: Family Medical Leave'. Do you want to pull that text too before answering?"

pdfs

Parsing PDFs for Graph RAG is notoriously tricky because standard PDF parsers throw away the exact thing you need most: visual structure.

When a standard parser converts a PDF to plain text, titles, subsections, tables, and paragraphs collapse into a single giant wall of text. The deterministic parent-child extraction we discussed earlier breaks down because you lose the visual cues of what is a "Header" vs. what is a "Paragraph."

To make Graph RAG work with PDFs without using arbitrary text overlaps, you must use a Layout-Aware PDF Parser to reconstruct the document's physical hierarchy before sending it to Neo4j.

1. The Stack: Layout-Aware Parsing
Instead of standard tools like PyPDF, you should use parsers that use object detection or deep layout analysis to identify structural elements (Headers, Text, Tables). Great options include:

unstructured (Excellent native integration with LangChain)

Marker / PyMuPDF4LLM (Converts PDFs directly into clean Markdown)

LlamaParse (Cloud-based, exceptional at preserving complex hierarchies and tables)

For policy PDFs, converting the PDF into Markdown first is the path of least resistance because Markdown preserves structural metadata (like # and ##) natively.

2. Implementation: From PDF to Linked Paragraphs
Here is how to ingest a policy PDF, extract its visual structure, chunk it cleanly without overlaps, and link it into a parent-child graph layout using LangChain and Python.

Python
import os
import uuid
from langchain_community.document_loaders import PyMuPDF4LLM
from langchain_text_splitters import MarkdownHeaderTextSplitter
from langchain_community.graphs import Neo4jGraph

# 1. Connect to Neo4j
graph = Neo4jGraph()

# 2. Extract PDF straight into clean Markdown text
# PyMuPDF4LLM detects fonts/sizes to guess headers (#, ##) automatically
pdf_path = "company_travel_policy.pdf"
md_docs = PyMuPDF4LLM(pdf_path).load()
full_markdown_text = md_docs[0].page_content 

# 3. Split based on visual hierarchy (Headers) instead of character counts
headers_to_split_on = [
    ("#", "Document"),
    ("##", "Section"),
    ("###", "SubSection")
]
markdown_splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers_to_split_on)
semantic_chunks = markdown_splitter.split_text(full_markdown_text)

# 4. Ingest into Neo4j, creating structural relationships
doc_id = f"doc_{str(uuid.uuid4())[:6]}"
doc_title = os.path.basename(pdf_path)

# Track previous paragraph to link sequential chunks together
prev_paragraph_id = None

for chunk in semantic_chunks:
    section_title = chunk.metadata.get("Section", "General Rules")
    subsection_title = chunk.metadata.get("SubSection", "Standard Protocol")
    paragraph_text = chunk.page_content
    paragraph_id = f"p_{str(uuid.uuid4())[:8]}"
    
    # Cypher query to build the layout hierarchy
    cypher_query = """
    MERGE (d:Document {id: $doc_id}) SET d.title = $doc_title
    MERGE (s:Section {title: $section_title})
    MERGE (sub:SubSection {title: $subsection_title})
    MERGE (p:Paragraph {id: $p_id}) SET p.text = $p_text
    
    // Draw structural lines
    MERGE (d)-[:HAS_SECTION]->(s)
    MERGE (s)-[:HAS_SUBSECTION]->(sub)
    MERGE (sub)-[:HAS_PARAGRAPH]->(p)
    """
    
    graph.query(cypher_query, {
        "doc_id": doc_id, "doc_title": doc_title,
        "section_title": section_title, "subsection_title": subsection_title,
        "p_id": paragraph_id, "p_text": paragraph_text
    })
    
    # Replaces 'Overlap': Link sequential paragraphs together
    if prev_paragraph_id:
        link_query = """
        MATCH (prev:Paragraph {id: $prev_id}), (curr:Paragraph {id: $curr_id})
        MERGE (prev)-[:NEXT_PARAGRAPH]->(curr)
        """
        graph.query(link_query, {"prev_id": prev_paragraph_id, "curr_id": paragraph_id})
        
    prev_paragraph_id = paragraph_id
3. Handling the "PDF Table" Nightmare
Policies are packed with tables (e.g., Expense allowance limits by tier). If you split text blindly by character counts, tables turn into useless gibberish.

When using a layout parser like PyMuPDF4LLM or unstructured, tables are typically converted into HTML string format (<table>...</table>) or clean Markdown tables inside the text block.

The Graph RAG Way to Store Tables:
Do not treat a table like a standard paragraph. Give it its own identity in your graph:

(:SubSection) ──[:HAS_TABLE]──> (:Table {html_content: "...", summary: "..."})
Extract: Let your parser isolate the table object.

Summarize: Run the table through an LLM to generate a plain-text summary (e.g., "Table showing standard hotel allowance caps per city tier").

Embed & Store: Compute the vector embedding on the summary, but store both the summary and the raw HTML/Markdown string on the graph node.

Query: When the vector search matches the summary, your LangGraph retriever hands the raw, perfectly formatted table matrix to the LLM.

4. The Payoff in LangGraph
Because your parser reconstructed the PDF layout, your LangGraph retrieval agent can adapt to messy user questions.

If a user asks: "What does section 4 say?", standard vector RAG fails because "Section 4" doesn't have a distinct semantic vector semantic profile. But your Graph RAG agent can circumvent vector search entirely and run a deterministic structural query:

Cypher
MATCH (:Document {title: "travel_policy.pdf"})-[:HAS_SECTION]->(s:Section)
WHERE s.title CONTAINS "Section 4"
MATCH (s)-[:HAS_SUBSECTION]->()-[:HAS_PARAGRAPH]->(p:Paragraph)
RETURN p.text ORDER BY p.id
This structural fallback guarantees that your agent finds the exact data required, even if the user didn't use the precise semantic keywords found inside the PDF paragraphs.

Are you planning on updating these policies frequently, or is this a static set of documents? Knowing this can help structure how we handle document versions and [:SUPERSEDES] relationships.
