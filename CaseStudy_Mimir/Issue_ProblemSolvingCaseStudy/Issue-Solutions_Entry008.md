# RAG Planning and Implementation
## Overview

This document outlines the comprehensive planning and implementation strategy for integrating Retrieval-Augmented Generation (RAG) capabilities into Mimir, an educational AI tutor chatbot. You will find detailed technical specifications, implementation roadmaps, and maintenance strategies designed to transform a static knowledge base into a dynamic, current information system.

**What You Will Find:**

The model used for Mimir is a fine-tuned version of `Phi3.5`, which has a pretraining cutoff of October 2023. The purpose of Mimir, aside from being a live demo for my portfolio, is to be an example of a full product and therefore my ability to make deployment-ready AI products. A significant knowledge cutoff greatly reduces Mimir's ability to provide effective responses on dynamic facts, such as test information queries. In cases like the SAT where structure has changed, or in other tests where times may change, Mimir will only have out-of-date information.

**Why This Documentation Exists:**

With this issue identified, I concluded that to be a production standard tutor chatbot Mimir needs up-to-date information in a form that can be modified if needed for future changes, as opposed to training the fine-tuned version on new information. This option is more flexible, allowing for out-of-date information to be removed or updated over time rather than needing to spend significant time retraining `Mimir-Phi3.5`. This document provides the complete framework for implementing this solution.

**Technical Architecture Decisions:**

I have opted to use a database approach, hosting the documents in a Hugging Face dataset, as opposed to integrating a web search API like Brave API. I made this decision for two reasons. First, the cost for web search API access, with applicable query limits, is out of scope for a demo. Secondly, and most importantly, web search API requires additional training and refinement of sources or search parameters, which is not ideal for a single developer. In the interest of being realistic with my resources and time availability for this project, a static database is more suitable.

**Document Structure and Purpose:**

This documentation contains a comprehensive Definition of Done (DoD) that establishes clear completion criteria, a detailed data storage and repository management strategy using `jdesiree112/Mimir_RAG_Database`, and an extensive document curation plan spanning seven phases of educational content. You will find maintenance schedules, quality control procedures, and metadata strategies that ensure the RAG system remains current and reliable over time. The included processing pipeline specifications address the technical challenges of converting PDF documents to searchable text while preserving document integrity and traceability. Finally, the document provides update schedules and maintenance protocols designed to keep educational information current without requiring constant manual oversight.

This comprehensive approach ensures that Mimir can provide accurate, up-to-date educational guidance while demonstrating enterprise-level planning and implementation capabilities for portfolio evaluation.

### User Story
As a student or educator, I want to use Mimir to research, study and prepare for educational purposes with up-to-date responses.

### DoD

| Activity | Description |
| :-: | :- |
| Code Complete | All coding tasks are complete, with needed changes implemented. The LangChain components have been created in a dedicated folder, integrated into relevant agents, and appropriate modules have been imported as needed. Prompt templates have been created or adapted for the RAG pipeline. |
| Unit Testing Passed | All RAG components are functioning as expected, with relevant information chunks being passed to the response agent. |
| Functional Requirements Met | The code results in effective information querying, retrieving, embedding and integration. |
| Design and Architecture | The design and architecture are compatible with the deployment environment, Hugging Face Spaces, and are functional with existing LangChain and LangGraph orchestration. Agents have been updated to incorporate RAG results. The document library has been curated, either through custom documents or open source/non-restricted documents that adhere to copyright and AI use legislation. |
| Code Review | Code has been reviewed and meets the standard for this project.|
| Documentation Updated | This Issue and Solution report has been updated to include results and reflections. API Documentation has been revised. |
| Security Measures and Considerations | The new system has been checked for vulnerabilities |
| User Interface Finalized | The user interface should have no changes. It has been verified that the user interface is not affected. |
| Regression Testing | Mimir has been tested to ensure no regression of the application function. |
| Deployment Ready | The code is ready to deploy in Hugging Face Spaces. |
| Technical Debt Evaluation | The code and automated evaluations and/or metrics tracking have been assessed, ensuring no unnecessary technical debt. All needed revisions have been made. |

### Maintenance and Response Handling
This segment of Entry008 focuses on identifying areas where dynamic information is needed to satisfy user needs. The items will be screened annually, with needed updates or removals applied at that time. Data may also be updated if I am made aware of deprecated data through evaluation processes, user feedback or other means. To handle the possibility of data becoming out of date prior to the annual update, I plan to add the current day's date to prompts by means of the datetime Python library and f-string prompt templating. For awareness, document chunks will be annotated with the date they were last modified. The `CORE_IDENTITY` prompt template, which is applied regardless of prompt context, will be updated to direct models to consider the date discrepancy and inform users to verify rapidly changing facts if the difference is six months or longer.

### Document Curation
For document curation, copyright law and AI legislation must be considered. When allowable by CC0, Open Source, or otherwise non-restrictive licensing, documents will be sourced. In cases where a document cannot be found to suit a specific fact, I will create a document that explains the current information based on my own research through combined open sources.

Yes, you'll need to convert the PDFs to text format for optimal use with LangChain's `HuggingFaceDatasetLoader`. Here's the updated data storage section:

## Data Storage & Repository Management

### Hugging Face Dataset Repository
The educational knowledge base will be hosted in the Hugging Face dataset repository `jdesiree112/Mimir_RAG_Database`. This repository will serve as the centralized storage for all curated documents and will be directly accessible through LangChain's `HuggingFaceDatasetLoader`.

#### PDF Processing & Text Extraction
Since the source documents will primarily be PDFs, a preprocessing pipeline will be implemented to extract and structure the text content:

**PDF Processing Tools:**
- `PyPDF2` or `pdfplumber` for standard PDF text extraction
- `unstructured` library for complex document layouts
- `pymupdf` (fitz) for advanced PDF parsing when needed

**Extraction Process:**
1. Download original PDF documents
2. Extract text content while preserving structure
3. Clean and format extracted text
4. Split into logical chunks (by section, page, or semantic meaning)
5. Generate metadata including original PDF information
6. Store processed text in dataset repository

#### Repository Structure
- **Repository Name**: `jdesiree112/Mimir_RAG_Database`
- **Access Level**: Public (for portfolio demonstration purposes)
- **File Format**: JSON Lines (.jsonl) for optimal streaming and processing
- **Version Control**: Git-based versioning through Hugging Face Hub
- **Content Storage**: Extracted text content, not original PDFs

#### Data Schema
Each processed document entry will follow this standardized schema:
```json
{
    "id": "unique_document_identifier",
    "title": "Document Title",
    "text": "Extracted text content from PDF",
    "source": "Source organization/website",
    "original_filename": "original_pdf_filename.pdf",
    "page_number": 1,
    "chunk_id": "doc_001_chunk_001",
    "publication_date": "YYYY-MM-DD",
    "last_modified": "YYYY-MM-DD",
    "geographic_scope": "national|state|district",
    "grade_levels": ["9-12", "undergraduate"],
    "subject_areas": ["testing", "financial_aid", "etc"],
    "document_type": "policy|guide|FAQ|data",
    "expiration_date": "YYYY-MM-DD",
    "metadata": {
        "extraction_method": "pdfplumber|pypdf2|unstructured",
        "original_file_size": "size_in_bytes",
        "text_length": "character_count",
        "language": "en",
        "reliability_score": "1-10"
    }
}
```

#### Processing Pipeline
The document processing workflow will include:
1. **PDF Collection**: Gather PDFs from official sources
2. **Text Extraction**: Convert PDFs to clean text using appropriate tools
3. **Chunking Strategy**: Split documents into semantically meaningful chunks (512-1024 tokens)
4. **Metadata Enrichment**: Add relevant tags, dates, and classification information
5. **Quality Control**: Verify extraction accuracy and text readability
6. **Dataset Upload**: Push processed documents to Hugging Face repository
7. **Version Control**: Track changes and maintain document lineage

#### Repository Management
- **Upload Process**: Automated pipeline using Hugging Face `datasets` library
- **Version Control**: Git-based tracking of all document updates
- **Access Control**: Public read access for demonstration purposes
- **Backup Strategy**: Local storage of both original PDFs and processed text
- **Integration**: Direct compatibility with `HuggingFaceDatasetLoader` for RAG pipeline

#### Document To-Do List
# Document Curation To-Do List for Educational RAG System

##### Phase 1: Core Testing & Assessment Documents

###### Standardized Test Information
- [ ] Collect official test guides for SAT, ACT, PSAT from College Board and ACT.org
- [ ] Download current score ranges and percentile charts for all major tests
- [ ] Gather test date calendars for 2025-2026 academic year
- [ ] Compile fee schedules and fee waiver information
- [ ] Collect accommodation guidelines for students with disabilities

###### State-Specific Assessment Documentation
- [ ] Create state-by-state folders for all 50 states + DC
- [ ] Download official testing schedules from each state department of education
- [ ] Collect sample questions and practice materials for each state test
- [ ] Gather testing window information (when tests are administered)
- [ ] Document opt-out policies where applicable
- [ ] Collect scoring guides and proficiency level descriptions

###### AP & Advanced Course Materials
- [ ] Download current AP Course Descriptions for all 38+ courses
- [ ] Collect AP exam schedules and registration deadlines
- [ ] Gather scoring distributions and college credit policies
- [ ] Document digital vs paper format decisions by subject
- [ ] Collect sample questions and released exams
- [ ] Gather IB program information and comparison materials

##### Phase 2: College Admissions & Financial Aid

###### Admissions Requirements
- [ ] Create spreadsheet of test-optional colleges (update monthly)
- [ ] Collect Common Application updates and essay prompts
- [ ] Document application deadlines by admission type (EA, ED, RD)
- [ ] Gather admissions statistics (acceptance rates, average scores)
- [ ] Collect demonstrated interest policies and requirements

###### Financial Aid Information
- [ ] Download current FAFSA forms and instructions
- [ ] Collect federal aid eligibility requirements and amounts
- [ ] Gather state-specific aid programs documentation
- [ ] Document work-study opportunities and requirements
- [ ] Collect scholarship databases and application timelines
- [ ] Gather student loan information and interest rates

##### Phase 3: Academic Policies & Regulations

###### Graduation Requirements
- [ ] Compile state graduation requirements by subject area
- [ ] Document credit hour requirements and conversion policies
- [ ] Collect dual enrollment policies and credit transfer rules
- [ ] Gather career pathway requirements by state
- [ ] Document alternative assessment options for graduation

###### Student Rights & Policies
- [ ] Collect Title IX policies for educational institutions
- [ ] Document special education rights and IEP/504 processes
- [ ] Gather academic integrity policies and consequences
- [ ] Collect attendance and truancy law summaries by state
- [ ] Document student privacy rights (FERPA guidelines)

##### Phase 4: Career & Technical Education

###### Certification Programs
- [ ] Document industry certification requirements (Microsoft, Adobe, Google, etc.)
- [ ] Collect apprenticeship program information by trade and location
- [ ] Gather career pathway maps for major industries
- [ ] Document work-based learning requirements and opportunities
- [ ] Collect salary data for entry-level positions by field

###### STEM & Technology Education
- [ ] Compile programming language learning resources and current trends
- [ ] Document STEM competition rules and deadlines
- [ ] Collect laboratory safety protocols and updates
- [ ] Gather information on emerging technology in education
- [ ] Document maker space and technology access programs

##### Phase 5: Health, Safety & Student Support

###### Mental Health Resources
- [ ] Compile crisis hotline information by state/region
- [ ] Collect school counseling availability and procedures
- [ ] Document mental health first aid resources for educators
- [ ] Gather stress management and wellness program information
- [ ] Collect suicide prevention resources and protocols

###### Health & Safety Guidelines
- [ ] Document current health screening requirements
- [ ] Collect vaccination requirements by state for school attendance
- [ ] Gather emergency procedures and safety protocols
- [ ] Document food allergy management policies
- [ ] Collect sports/activity participation health requirements

##### Phase 6: Current Events & Social Issues

###### Civic Education
- [ ] Collect voter registration information for 18+ students
- [ ] Document civic engagement opportunities and requirements
- [ ] Gather information on student activism rights and limitations
- [ ] Collect community service graduation requirements by district
- [ ] Document mock election and civics competition resources

###### Environmental & Social Justice Education
- [ ] Compile climate change educational resources and current data
- [ ] Collect diversity, equity, inclusion policy updates
- [ ] Document social justice curriculum guidelines and resources
- [ ] Gather environmental science project ideas and competitions

##### Phase 7: Technology & Digital Learning

###### Educational Technology
- [ ] Document LMS platform updates (Canvas, Google Classroom, etc.)
- [ ] Collect AI policy guidelines for academic use
- [ ] Gather digital citizenship curriculum and resources
- [ ] Document cybersecurity education for students
- [ ] Collect device management and BYOD policies

###### Online Learning Resources
- [ ] Compile virtual course availability and requirements
- [ ] Document homeschool regulation updates by state
- [ ] Collect online safety protocols and guidelines
- [ ] Gather accessibility requirements for digital content

##### Document Organization & Metadata Strategy

###### Folder Structure
- [ ] Create /Testing-Assessment directory structure
- [ ] Create /College-Admissions directory structure
- [ ] Create /Academic-Policies directory structure
- [ ] Create /Career-Technical-Ed directory structure
- [ ] Create /Health-Safety-Support directory structure
- [ ] Create /Current-Events-Civics directory structure
- [ ] Create /Technology-Digital-Learning directory structure

###### Metadata Requirements for Each Document
- [ ] Document title and description
- [ ] Source organization and author
- [ ] Publication/update date
- [ ] Geographic scope (national, state-specific, district-specific)
- [ ] Grade level relevance (K-5, 6-8, 9-12, undergraduate)
- [ ] Subject area tags (math, English, science, etc.)
- [ ] Document type (policy, guide, FAQ, data, etc.)
- [ ] Expiration date or next review date
- [ ] Related documents and cross-references

##### Update Schedule & Maintenance

###### Weekly Updates Required
- [ ] Test registration deadlines and date changes
- [ ] Scholarship application deadlines
- [ ] Current events affecting education policy
- [ ] Health and safety guideline changes

###### Monthly Updates Required
- [ ] College admissions requirement changes
- [ ] Financial aid amount and eligibility updates
- [ ] State policy changes and new legislation
- [ ] Technology platform updates and new features

###### Seasonal Updates Required
- [ ] Testing calendars for upcoming academic year
- [ ] Graduation requirements updates (typically summer)
- [ ] Course catalog changes and new offerings
- [ ] Summer program and internship opportunities

###### Annual Updates Required
- [ ] Comprehensive policy review of all documents
- [ ] Statistical data updates (graduation rates, test scores, etc.)
- [ ] Program evaluation and resource effectiveness
- [ ] Archive outdated materials and create historical record

##### Quality Control Checklist

###### Before Adding Documents
- [ ] Verify source credibility - official education departments only
- [ ] Check publication date - no older than 2 years unless historical
- [ ] Confirm geographic relevance - match to intended audience
- [ ] Review for bias - stick to factual, policy-based content
- [ ] Test accessibility - ensure documents are readable/searchable

###### Ongoing Quality Assurance
- [ ] Monthly broken link checks for online resources
- [ ] Quarterly content audit for outdated information
- [ ] Annual user feedback review and integration
- [ ] Bi-annual competitor analysis of similar knowledge bases

---

## Updated and Reflects will be written here after full implmentation ad completiong per estabilished DoD Documentaiton

---
