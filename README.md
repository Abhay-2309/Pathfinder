# Intelligent Document Analyst for Adobe "Connecting the Dots" Challenge

## Project Overview

This project is a submission for the **Adobe "Connecting the Dots" Hackathon**. It is a comprehensive, persona-driven document intelligence system designed to reimagine how users interact with PDFs. The solution acts as an intelligent research companion that first understands the structure of documents and then extracts and prioritizes information based on a user's specific role and goals.

The system is built to be a generic, robust, and efficient offline tool, adhering strictly to the competition's constraints.

### Core Features
*   **Hierarchical Document Structuring (Round 1A):** Automatically extracts a clean, structured outline (Title, H1, H2, H3) from PDF documents.
*   **Persona-Driven Relevance Ranking (Round 1B):** Analyzes a collection of documents to identify and rank sections most relevant to a specific user persona and their "job-to-be-done."
*   **Granular Sub-Section Analysis:** Pinpoints and extracts the most precise and relevant sentences or paragraphs within high-importance sections.
*   **100% Offline Execution:** All models and dependencies are self-contained within a Docker image, requiring no internet access during runtime.
*   **Optimized for CPU Performance:** Built to run efficiently on a standard CPU architecture within the specified time and resource limits.

---

## Architecture & Technical Approach

Our solution is built as a two-stage pipeline, directly corresponding to the challenges in Round 1A and Round 1B.

### Module 1: Document Structuring Engine (Round 1A)

This module is the foundation of our system. It parses PDFs to create a structured outline.

*   **PDF Parsing:** We use the **`PyMuPDF` (fitz)** library. It was chosen for its exceptional speed, low memory footprint, and its ability to provide rich metadata for each text block, including font size, font name (for bold detection), and precise positional coordinates.

*   **Heading Detection Logic:** To avoid the pitfall of relying solely on font size, we implemented a **rule-based heuristic scoring system**. For each text block in a document, we calculate a "heading score" based on a weighted combination of features:
    *   **Relative Font Size:** Larger font size compared to the document's most common body text size.
    *   **Font Weight:** Detection of **bold** text from the font name.
    *   **Text Case:** All-caps text is given a higher score.
    *   **Positional Cues:** Text centered on the page or near the top is prioritized.
    *   **Numbering Patterns:** We use regular expressions to identify patterns like "1.", "2.1.", "Chapter 3", etc.
*   **Hierarchy Assignment:** After scoring, we identify the document's Title. Then, we cluster the remaining heading scores to dynamically determine the thresholds for H1, H2, and H3 levels, ensuring a logical document structure is maintained.

### Module 2: Relevance Ranking Engine (Round 1B)

This module builds on the structured output from Module 1 to deliver persona-driven insights.

*   **Core Technology:** The engine is powered by **semantic search** using sentence embeddings. This allows us to understand the *meaning* of the text, not just keywords.

*   **Offline NLP Model:** We use the `all-MiniLM-L6-v2` sentence-transformer model. This model was carefully selected because it provides an excellent balance of:
    *   **Performance:** High accuracy for semantic similarity tasks.
    *   **Size:** Very lightweight (~80MB), easily fitting within the project constraints.
    *   **Speed:** Optimized for fast inference on CPUs.

*   **Ranking Process:**
    1.  **Query Formulation:** The user's `Persona` and `Job-to-be-done` are combined into a single, descriptive query string.
    2.  **Embedding Generation:** The query string and the text content of every section (identified in Module 1) are converted into high-dimensional vectors (embeddings) using the offline model.
    3.  **Relevance Scoring:** We calculate the **Cosine Similarity** between the user query's embedding and each section's embedding. This score (from 0 to 1) represents how semantically relevant the section is to the user's need.
    4.  **Section Ranking:** The sections from all documents are ranked in descending order of their relevance score to produce the final `Importance_rank`.
    5.  **Sub-Section Analysis:** For the top-ranked sections, we perform a more granular analysis. The section's text is split into paragraphs/chunks, and each chunk is re-ranked against the user query. The highest-scoring chunks are returned as the "Refined Text," providing the user with the most potent and direct information.

---

## Tech Stack

*   **Language:** Python 3.9
*   **PDF Processing:** `PyMuPDF`
*   **NLP/Machine Learning:** `sentence-transformers`, `torch`, `scikit-learn`
*   **Containerization:** Docker

---

## Setup and Execution Instructions

The entire solution is packaged in a Docker container for easy and consistent execution.

### Prerequisites
*   [Docker](https://www.docker.com/get-started) must be installed and running on your system.

### Step 1: Build the Docker Image

Navigate to the project's root directory (where the `Dockerfile` is located) and run the following command to build the image. This will install all dependencies and package the offline model.

```bash
docker build --platform linux/amd64 -t adobe-hackathon-solution .
```

### Step 2: Prepare Input Data

Place all the PDF files you want to process inside the `input/` directory in the project's root.

```
/
├── input/
│   ├── document1.pdf
│   └── document2.pdf
└── ...
```

### Step 3: Run the Solution

Execute the following command to run the container. The script will automatically process all PDFs from the `input` folder and generate the corresponding JSON output in the `output` folder.

*   The `--rm` flag automatically removes the container when it finishes.
*   The `-v` flags mount the local `input` and `output` directories into the container.
*   The `--network none` flag ensures the solution runs completely offline, as per the rules.

```bash
docker run --rm \
  -v $(pwd)/input:/app/input \
  -v $(pwd)/output:/app/output \
  --network none \
  adobe-hackathon-solution
```

After execution, the `output/` directory will be populated with the resulting JSON files.

---

## Project Structure

```
.
├── Dockerfile
├── README.md
├── approach_explanation.md
├── requirements.txt
├── input/                  # (Create this) Place input PDFs here
├── output/                 # (Create this) Output JSONs will appear here
└── src/
    ├── main.py             # Main execution script
    ├── document_parser.py  # Module for PDF structuring (Round 1A)
    ├── relevance_engine.py # Module for relevance ranking (Round 1B)
    └── models/             # Contains the downloaded offline sentence-transformer model
```
