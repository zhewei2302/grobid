# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GROBID (GeneRation Of BIbliographic Data) is a machine learning library for extracting, parsing, and restructuring raw scientific documents (primarily PDFs) into structured XML/TEI encoded documents. Current version: 0.8.3-SNAPSHOT.

## Build Commands

```bash
# Standard build (tests skipped by default)
./gradlew clean install

# Build with tests
./gradlew clean install test

# Run development service on port 8070
./gradlew run

# Run specific test
./gradlew test --tests TestClassName

# Generate JaCoCo coverage report
./gradlew codeCoverageReport

# Build fat JAR for deployment
./gradlew shadowJar
```

**Requirements:** OpenJDK 21+

## Training Model Commands

```bash
# Simple model trainers
./gradlew train_name_header
./gradlew train_name_citation
./gradlew train_affiliation_address
./gradlew train_date
./gradlew train_citation
./gradlew train_reference_segmentation
./gradlew train_figure
./gradlew train_table
./gradlew train_funding_acknowledgement

# Complex trainers (with flavor variants)
./gradlew train_header
./gradlew train_header_article_light
./gradlew train_segmentation
./gradlew train_fulltext

# End-to-end evaluation
./gradlew jatsEval -Pp2t=/path/to/goldenSet
./gradlew teiEval -Pp2t=/path/to/goldenSet
```

## Project Structure

```
grobid/
├── grobid-core/        # Main library: extraction, parsing, ML models
├── grobid-service/     # REST API (Dropwizard, port 8070)
├── grobid-trainer/     # Model training utilities
├── grobid-home/        # Resources, pre-trained models, config
└── doc/                # MkDocs documentation
```

## Architecture

### Module Responsibilities

- **grobid-core**: Core extraction logic, ML engines, feature extraction, document processing, TEI formatting. Entry point: `org.grobid.core.main.batch.GrobidMain`
- **grobid-service**: REST API using Dropwizard 4.0.13, Guice DI, Prometheus metrics. Entry point: `org.grobid.service.main.GrobidServiceApplication`
- **grobid-trainer**: Training runners and evaluation tools. Main class: `org.grobid.trainer.TrainerRunner`
- **grobid-home**: Static resources, lexicons, native libraries (Wapiti), configuration files

### Key grobid-core Packages

- `engines/` - Parsing engines (HeaderParser, CitationParser, FullTextParser, etc.)
- `document/` - Document processing and TEI formatting
- `features/` - ML feature extraction classes
- `jni/` - JNI wrappers for native libraries (Wapiti CRF)
- `layout/` - Layout tokens and PDF coordinate handling
- `lexicon/` - Dictionary lookups

### Cascade of Sequence Labeling Models

GROBID uses a hierarchical model chain where each model handles specific extraction tasks:

1. **Segmentation** → identifies document zones (header, body, references, etc.)
2. **Header** → extracts title, authors, affiliations, abstract
3. **Fulltext** → structures body content (paragraphs, sections, figures, tables)
4. **Citation** → parses bibliographic references
5. Specialized models: Date, Affiliation-Address, Name, Figure, Table

### ML Engine Options

- **Wapiti (default)**: CRF-based, fast, feature-engineered
- **DeLFT (optional)**: Deep Learning with layout features, requires Python/JEP

## Configuration

Main config: `grobid-home/config/grobid.yaml`

Key settings:
- `grobidHome`: Path to grobid-home resources
- `pdfalto.memoryLimitMb`: PDF parsing memory limit (default 6096 MB)
- `pdfalto.timeoutSec`: Processing timeout (default 120s)
- `concurrency`: Max parallel request threads (default 10)
- Model-specific engine selection (wapiti/delft)

## Service API

- **Main API**: Port 8070, endpoints at `/api/*`
- **Admin console**: Port 8071
- **Metrics**: `/metrics/prometheus`

HTTP response codes:
- 200: Success
- 204: Success but no content extracted
- 400: Bad request
- 500: Internal error
- 503: Service unavailable (thread pool exhausted)

## Testing

- Framework: JUnit 5 with EasyMock/Mockito
- Test isolation: `forkEvery = 1`
- Coverage: JaCoCo integrated

## Docker

```bash
# CRF models only (lighter)
docker build -f Dockerfile.crf -t grobid:latest .

# With Deep Learning support
docker build -f Dockerfile.delft -t grobid:full .

docker run -p 8070:8070 grobid:latest
```

## Key Dependencies

- PDF processing: Apache PDFBox 2.0.33
- Web framework: Dropwizard 4.0.13
- XML: Saxon-HE, Xerces, XOM 1.3.9
- ML: Wapiti (bundled), DeLFT (optional)
- Python integration: JEP 4.2.2 (for Deep Learning)

## Design Principles

1. **Layout tokens, not just text**: Features exploit visual/positional information (font, style, coordinates)
2. **Quality over quantity**: Small, manually-annotated training sets from actual PDFs
3. **PDF coordinate tracking**: Bounding boxes synchronized with extracted content for visualization
4. **Service-oriented**: Web service is preferred over batch processing for production use
