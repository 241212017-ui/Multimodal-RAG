# Multimodal RAG

A Retrieval-Augmented Generation system that can understand and retrieve information from text, images, diagrams, charts, and tables inside PDF documents.

Traditional RAG systems mainly work with text. That becomes a limitation when important information is stored inside a diagram, chart, or table rather than written as normal paragraphs.

This project extends the traditional RAG pipeline by processing multiple types of document content and creating separate retrieval indexes for each modality. When a user asks a question, the system determines which type of content is most relevant, retrieves the required information, and generates an answer using the retrieved context.

For example, a text-based RAG system may understand a paragraph describing an architecture, but it cannot directly understand the architecture diagram itself. This system extracts the diagram, generates a description of its visual content, indexes that description, and makes it searchable.

The goal is to make document question answering more useful for real-world documents such as research papers, annual reports, technical documentation, business reports, and academic publications.

---

## Overview

The system processes a PDF in several stages.

First, the document is parsed to identify text, images, and tables. Each content type is processed separately.

Text is divided into meaningful chunks and converted into embeddings.

Images such as diagrams, flowcharts, graphs, and figures are extracted from the PDF. A vision-language model analyzes each image and generates a textual description of its content.

Tables are extracted as structured data. An LLM converts the table into a searchable natural-language description while preserving important values and relationships.

The resulting information is stored in separate FAISS indexes.

When a question is submitted, a query router determines whether the question is related to text, images, tables, or multiple modalities. The appropriate indexes are searched, the results are combined, and the retrieved context is passed to the language model to generate the final answer.

---

## Why Multimodal RAG

Important information in modern documents is not limited to paragraphs.

A research paper may contain an architecture diagram that explains how different components interact. An annual report may contain financial information only inside tables. A technical document may use a flowchart to explain a process.

A traditional text-only RAG system may fail to retrieve this information because the visual or structured content is not represented in its text index.

This project addresses that limitation by treating different document modalities as first-class sources of information.

| Query                                   | Traditional Text RAG                  | Multimodal RAG                        |
| --------------------------------------- | ------------------------------------- | ------------------------------------- |
| Explain the authentication flow         | Can answer if described in text       | Can answer                            |
| What does the architecture diagram show | Usually cannot answer                 | Can analyze the diagram               |
| What was the Q4 revenue                 | Depends on whether it appears in text | Can retrieve the value from a table   |
| Describe the flowchart in section 3     | Cannot directly understand the image  | Can analyze the extracted figure      |
| Summarize the key findings              | Mostly uses text                      | Can combine text, figures, and tables |

---

## System Architecture

```text
                         PDF Document
                              |
                              v
                  Multimodal Document Parser
                              |
              +---------------+---------------+
              |               |               |
              v               v               v
            Text            Images          Tables
              |               |               |
              v               v               v
        Text Chunking     Vision Model     Table Processing
              |          Image Captioning   LLM Description
              |               |               |
              +---------------+---------------+
                              |
                              v
                    Embedding Generation
                              |
              +---------------+---------------+
              |               |               |
              v               v               v
          Text FAISS      Image FAISS      Table FAISS
             Index           Index            Index
              |               |               |
              +---------------+---------------+
                              |
                              v
                        Query Router
                              |
                +-------------+-------------+
                |             |             |
              Text          Image         Table
                |             |             |
                +-------------+-------------+
                              |
                              v
                    Multi-Modal Retriever
                              |
                              v
                    Retrieved Context
                              |
                              v
                         LLM Generator
                              |
                              v
                       Final Answer
```

---

## How the System Works

### 1. PDF Parsing

The PDF is processed using `pdfplumber`.

The parser identifies:

* Text blocks
* Images
* Tables
* Page information
* Extracted document elements

Images are saved as image files and tables are stored in structured formats for further processing.

### 2. Text Processing

Text is divided into smaller chunks so that relevant sections can be retrieved efficiently.

Each chunk is converted into an embedding using the `all-MiniLM-L6-v2` model.

The embeddings are stored in a FAISS index that supports fast similarity search.

### 3. Image Processing

Images extracted from the PDF can contain important information such as:

* Architecture diagrams
* Flowcharts
* Graphs
* Charts
* Technical figures
* Process diagrams

The system sends these images to a vision-capable model such as `gpt-4o-mini`.

The model generates a textual description of the image.

For example:

```text
The diagram shows a three-stage data processing pipeline.
Raw data is first collected by the ingestion layer, then
processed by the transformation layer, and finally stored
in the database layer.
```

This description can then be embedded and stored in the image FAISS index.

The original image remains associated with its generated description so that the system can identify where the information originated.

### 4. Table Processing

Tables are extracted from the PDF as structured rows and columns.

Instead of treating the table as ordinary text, the system uses an LLM to generate a meaningful description.

For example, a table containing quarterly revenue can be converted into information such as:

```text
The company reported revenue of 18 million dollars in Q1,
21 million dollars in Q2, 24 million dollars in Q3, and
27 million dollars in Q4.
```

This representation makes numerical information easier to retrieve using semantic search.

### 5. Separate Retrieval Indexes

The project maintains three FAISS indexes:

```text
Text Index
    |
    +-- Paragraph and document chunks

Image Index
    |
    +-- Generated descriptions of figures and diagrams

Table Index
    |
    +-- Generated descriptions of extracted tables
```

Keeping the indexes separate allows the system to search the most relevant modality instead of treating every document element as identical.

### 6. Query Routing

The query router determines what kind of information the user is requesting.

Examples:

```text
"What does the architecture diagram show?"
                    |
                    v
                 IMAGE
```

```text
"What was the company's Q4 revenue?"
                    |
                    v
                 TABLE
```

```text
"What is the company's strategy?"
                    |
                    v
                  TEXT
```

Some questions require multiple modalities.

For example:

```text
"Explain the data pipeline shown in Figure 2 and describe
the purpose of each component."
```

The system can retrieve both the figure description and relevant text surrounding the figure.

### 7. Multi-Modal Retrieval

The retriever searches the indexes selected by the query router.

The retrieved results are combined into a single context.

Each result maintains its modality so the generator knows whether the information came from:

* Text
* Image
* Table

This allows the final response to provide more transparent answers.

### 8. Answer Generation

The retrieved context is passed to the language model.

The generator creates the final answer while using only the retrieved information as supporting context.

The response can identify the source modality, making it easier to understand where the information came from.

For example:

```text
The authentication process begins with user credentials
being submitted to the authentication service. The service
then validates the credentials before issuing an access token.

Source: Architecture diagram and surrounding text.
```

---

## Models and Technologies

| Component            | Technology       | Purpose                                                |
| -------------------- | ---------------- | ------------------------------------------------------ |
| PDF parsing          | pdfplumber       | Extract text, images, and tables                       |
| Text embeddings      | all-MiniLM-L6-v2 | Generate text embeddings                               |
| Vector search        | FAISS            | Fast similarity search                                 |
| Image understanding  | gpt-4o-mini      | Generate descriptions of visual content                |
| Table processing     | OpenAI models    | Convert structured tables into searchable descriptions |
| Query routing        | LLM              | Determine relevant modality                            |
| Answer generation    | GPT model        | Generate the final response                            |
| Programming language | Python           | Application development                                |

The project is designed so that the vision and generation models can be replaced with other compatible models.

For local image processing, models such as LLaVA can also be used through Ollama.

---

## Key Features

### Multi-Modal Document Understanding

The system works with more than plain text. It can process text, images, diagrams, charts, and tables from the same PDF.

### Modality-Specific Retrieval

Separate FAISS indexes are maintained for different types of content.

This allows the retrieval process to focus on the most relevant source.

### Query Routing

The system analyzes the user's question before retrieval and determines whether it requires text, image, table, or combined retrieval.

### Vision-Based Image Understanding

Extracted figures can be analyzed using a vision-language model so that visual information becomes searchable.

### Table Understanding

Tables are converted into meaningful descriptions so that users can ask questions about numerical and structured information.

### Cross-Modality Retrieval

Some questions require information from more than one source. The retriever can combine relevant results from multiple indexes.

### Source-Aware Responses

The generated answer can identify whether supporting information came from text, an image, or a table.

### Interactive Question Answering

The system supports an interactive mode where multiple questions can be asked about the same indexed document.

---

## Example Questions

The system is designed to answer questions such as:

```text
What does the architecture diagram show?

Describe the authentication flow shown in Figure 3.

What was the company's Q4 revenue?

What were the year-over-year growth percentages?

Explain the data pipeline shown in the diagram.

Which component is responsible for data processing?

Summarize the main findings from the report.

Compare the values reported in Q3 and Q4.

What is the relationship between the components shown in Figure 2?
```

The system can use different modalities depending on the question.

---

## Project Structure

```text
04-multimodal-rag/
|
├── README.md
├── requirements.txt
├── .env.example
|
├── data/
│   ├── sample_docs/
│   │   └── Put PDF documents here
│   |
│   └── extracted/
│       ├── images/
│       │   └── Extracted PDF images
│       |
│       └── tables/
│           └── Extracted tables
|
├── src/
│   ├── multimodal_parser.py
│   │   └── Extracts text, images, and tables
│   |
│   ├── text_indexer.py
│   │   └── Creates the text FAISS index
│   |
│   ├── image_processor.py
│   │   └── Generates descriptions for images
│   |
│   ├── image_indexer.py
│   │   └── Creates the image FAISS index
│   |
│   ├── table_processor.py
│   │   └── Converts tables into descriptions
│   |
│   ├── table_indexer.py
│   │   └── Creates the table FAISS index
│   |
│   ├── query_router.py
│   │   └── Determines the required modality
│   |
│   ├── multi_retriever.py
│   │   └── Retrieves and combines results
│   |
│   └── generator.py
│       └── Generates the final answer
|
└── main.py
    └── Application entry point
```

---

## Requirements

Before running the project, make sure you have:

* Python 3.10 or newer
* An OpenAI API key
* A PDF containing meaningful text, images, diagrams, charts, or tables
* The required Python dependencies

Python 3.10 or newer is required because parts of the project use modern Python type-hint syntax.

---

## Installation

### Clone the Project

```bash
git clone <your-repository-url>
cd 04-multimodal-rag
```

### Create a Virtual Environment

On Windows:

```bash
python -m venv .venv
.venv\Scripts\activate
```

On Linux or macOS:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### Install Dependencies

```bash
pip install -r requirements.txt
```

### Configure the API Key

Create a `.env` file from the example configuration:

```bash
cp .env.example .env
```

Then add your OpenAI API key:

```env
OPENAI_API_KEY=your_api_key_here
```

On Windows, you can also create the file manually if the `cp` command is unavailable.

---

## Add a PDF

The `data/sample_docs/` directory is intentionally empty.

Add a PDF containing mixed content:

```text
data/
└── sample_docs/
    └── annual_report.pdf
```

For the best demonstration, use a document containing:

* Technical diagrams
* Flowcharts
* Charts
* Tables
* Normal paragraphs

A plain text PDF will work, but it will not demonstrate the main advantage of this project.

---

## Usage

### Ask a Single Question

```bash
python main.py --file data/sample_docs/annual_report.pdf \
               --query "What was Q4 revenue?"
```

### Ask About a Diagram

```bash
python main.py --file data/sample_docs/annual_report.pdf \
               --query "Describe the architecture diagram"
```

### Summarize the Document

```bash
python main.py --file data/sample_docs/annual_report.pdf \
               --query "Summarise the key findings"
```

### Interactive Mode

```bash
python main.py --file data/sample_docs/annual_report.pdf \
               --interactive
```

Interactive mode allows multiple questions to be asked about the same document.

### Skip Image Processing

Image captioning requires a vision model and may increase API usage.

For development, image processing can be disabled:

```bash
python main.py --file data/sample_docs/annual_report.pdf \
               --query "Summarise the key findings" \
               --skip-images
```

### Skip Images and Tables

For the fastest text-only development workflow:

```bash
python main.py --file data/sample_docs/report.pdf \
               --query "What is the company's strategy?" \
               --skip-images \
               --skip-tables
```

### Use Different Models

Models can be specified from the command line:

```bash
python main.py --file data/sample_docs/report.pdf \
               --query "Describe the architecture diagram" \
               --model gpt-4o \
               --vision-model gpt-4o
```

---

## Command Line Options

| Option           | Description                                     |
| ---------------- | ----------------------------------------------- |
| `--file`         | Path to the PDF document                        |
| `--query`        | Question to ask about the document              |
| `--model`        | Language model used for answer generation       |
| `--vision-model` | Vision model used for image understanding       |
| `--skip-images`  | Disable image processing                        |
| `--skip-tables`  | Disable table processing                        |
| `--interactive`  | Start an interactive question-answering session |

---

## Example Workflow

Suppose the PDF contains a company architecture diagram and a quarterly revenue table.

The user asks:

```text
What was Q4 revenue and how does the architecture
support the company's data processing system?
```

The query requires more than one type of information.

The system can retrieve:

```text
Table Index
    |
    +-- Q4 revenue information

Image Index
    |
    +-- Architecture diagram description

Text Index
    |
    +-- Supporting explanation from the document
```

These results are combined and provided to the generator.

The final answer can therefore use information from the table, diagram, and surrounding text instead of relying on a single text index.

---

## Cost Considerations

The main additional cost compared with a traditional text-only RAG system comes from processing images and using language models for table descriptions.

Image processing can require a vision model call for every extracted image.

For this reason, the project includes the `--skip-images` option.

A practical development workflow is:

```text
Development
    |
    +-- Use local embeddings
    +-- Skip image processing when unnecessary
    +-- Test text retrieval first
    +-- Process images only when required

Production
    |
    +-- Process each document once
    +-- Cache generated image descriptions
    +-- Cache table descriptions
    +-- Reuse existing embeddings
    +-- Use appropriate models based on cost and quality
```

Image and table descriptions should ideally be generated once and stored so that repeated questions do not trigger unnecessary processing.

For completely local experimentation, a vision model such as LLaVA through Ollama can be used instead of an API-based vision model.

---

## Multimodal RAG vs Basic RAG

This project builds on the concepts introduced by a traditional RAG pipeline.

| Feature                  | Basic RAG | Multimodal RAG     |
| ------------------------ | --------- | ------------------ |
| Text retrieval           | Yes       | Yes                |
| Image understanding      | No        | Yes                |
| Table understanding      | Limited   | Yes                |
| Vector indexes           | One       | Multiple           |
| Query routing            | No        | Yes                |
| Vision model             | No        | Yes                |
| Table processing         | No        | Yes                |
| Cross-modality retrieval | No        | Yes                |
| Source modality          | Text      | Text, image, table |
| System complexity        | Lower     | Higher             |
| API usage                | Lower     | Higher             |

The important difference is not simply adding more models.

The main architectural change is that the system recognizes that a document contains different types of information and gives each type an appropriate processing and retrieval strategy.

---

## What This Project Demonstrates

This project demonstrates several important concepts used in modern AI systems.

### Retrieval-Augmented Generation

Instead of asking an LLM to answer entirely from its internal knowledge, relevant information is retrieved from an external document and supplied as context.

### Vector Search

Document information is represented as embeddings and searched using similarity rather than exact keyword matching.

### Multimodal AI

Images and structured information are incorporated into a retrieval pipeline instead of treating the document as text only.

### Vision-Language Models

Visual information is converted into semantic descriptions that can participate in retrieval.

### Query Classification

The system determines what type of information a question requires before searching.

### Multiple Vector Indexes

Different modalities are stored in dedicated indexes so that retrieval can be targeted.

### Context Fusion

Results from different modalities can be combined before generation.

### Grounded Generation

The final response is generated from information retrieved from the source document.

---

## Limitations

The system is not perfect, and multimodal document processing introduces additional challenges.

Image descriptions depend on the quality of the vision model. Complex diagrams may be difficult to interpret correctly.

PDF table extraction can also be unreliable when tables have unusual formatting, merged cells, multiple headers, or complicated layouts.

The query router may occasionally select the wrong modality, especially for questions that require information from several parts of a document.

OCR quality can also affect results when information is contained in scanned PDFs or images.

The system therefore works best with digitally generated PDFs that contain clear text, well-structured tables, and reasonably readable visual content.

---

## Future Improvements

Several improvements can make the system more robust.

### Better Document Layout Understanding

Instead of processing content independently, a layout-aware parser could preserve relationships between headings, paragraphs, figures, captions, and tables.

### Better Image Retrieval

Image embeddings could be added alongside generated captions to improve retrieval of visually similar content.

### Hybrid Search

Combining semantic vector search with keyword-based search could improve retrieval of exact values, technical terms, and identifiers.

### Reranking

A reranking model could evaluate retrieved results and select the most relevant context before generation.

### Better Table Reasoning

Instead of converting tables only into prose, structured table querying could be introduced for more reliable numerical reasoning.

### Local Models

More components could be moved to local models to reduce API costs and improve privacy.

### Persistent Indexing

Document indexes could be stored permanently so that documents do not need to be processed again for every execution.

### Evaluation

A proper evaluation framework could measure:

* Retrieval accuracy
* Query routing accuracy
* Table question accuracy
* Image question accuracy
* Answer faithfulness
* End-to-end response quality

---

## Use Cases

This architecture can be applied to many types of real-world documents.

### Research Papers

Retrieve information from experiments, figures, tables, and technical diagrams.

### Financial Reports

Answer questions about revenue, growth, financial tables, and charts.

### Technical Documentation

Understand system architectures, workflows, diagrams, and implementation details.

### Business Reports

Combine written analysis with charts and structured data.

### Academic Documents

Search across text, figures, equations, and tables.

### Engineering Documents

Retrieve information from technical drawings, process diagrams, and specifications.

---

## Project Goals

The main goal of this project is to move beyond the limitations of text-only document retrieval.

A document should not be treated as a collection of paragraphs when its important information may exist in a diagram or table.

The system therefore follows a simple principle:

```text
Understand the document
        |
        v
Separate its modalities
        |
        v
Process each modality appropriately
        |
        v
Create searchable representations
        |
        v
Route the user's question
        |
        v
Retrieve relevant information
        |
        v
Combine the evidence
        |
        v
Generate a grounded answer
```

This makes the project a practical example of how multimodal models, vector databases, retrieval systems, and LLMs can be combined into a single document intelligence pipeline.

---

## Conclusion

Multimodal RAG extends the traditional Retrieval-Augmented Generation architecture by allowing information from text, images, and tables to participate in the retrieval process.

The system does not simply ask an LLM to understand an entire PDF. Instead, it builds a structured pipeline that extracts different content types, creates searchable representations, routes questions to relevant indexes, combines retrieved evidence, and generates an answer based on that evidence.

The result is a more capable document question-answering system that can work with the way information actually appears in modern documents.

---

## License

This project is intended for educational and research purposes. Add the license that matches your repository and intended usage.

## Author
RAZZAK

Developed as a project exploring multimodal document understanding, retrieval-augmented generation, vector search, and vision-language models.
