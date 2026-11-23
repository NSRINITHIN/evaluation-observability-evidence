# Evidence Pack - Environment Information

## System Information
- **Python Version**: 3.10.14
- **Platform**: Linux (Linux 4.19.0-gvisor x86_64)
- **Date**: 2026-11-23

## Project: Claude AI Engineer — Evaluation and Observability

Three systems evaluated for reliability, observability, and safeguards:

1. **System 1**: Validated, Routed Insurance Policy Extraction Pipeline
2. **System 2**: Schema-Enforced Two-Pass Mortgage Document Extraction  
3. **System 3**: Multi-Source Synthesis Supply Chain Risk Investigation

---

## Notes on Environment Constraints

- **System 1 & 2**: Requires Python 3.11+ for StrEnum support
  - **Resolution**: Added Python 3.10 compatibility shim to models.py
  - Both systems now run successfully with all tests passing
  
- **System 3**: Requires chromadb and sentence-transformers (heavy dependencies)
  - Installation started in background
  - Framework and fixtures available for offline evaluation

