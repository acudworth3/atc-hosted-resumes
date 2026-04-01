# Copilot Instructions for CV Pipeline

## Project Overview

This is an automated CV generation pipeline that creates multiple psychologically-optimized CV variants from YAML data. The system uses a Makefile-driven build process to transform YAML files → Python validation/generation → LaTeX compilation → PDF output, with automated testing to verify data completeness.

**Key purpose**: Users maintain a single source of truth (YAML data files) and automatically generate multiple CV variants for different roles, plus ATS-friendly plain text versions.

## Build, Test, and Lint Commands

### Build Commands

```bash
# Build all CV variants (default: software-developer, devops-engineer, cloud-engineer)
make all

# Build single variant
make software-developer
make devops-engineer
make cloud-engineer

# Generate ATS-friendly plain text versions
make ats-all
# Or individual ATS files: make output/ats/software-developer.txt

# Show all available targets
make help
```

### Test Commands

```bash
# Run data completeness tests (verifies all YAML data appears in PDFs)
make test

# Run test manually with detailed output
python3 scripts/test_data_completeness.py

# Check YAML syntax
python3 -m yaml < data/personal.yaml
```

### Cleanup

```bash
make clean  # Remove all generated PDFs and ATS text files
```

## Architecture

### Data Pipeline

```
YAML Files (data/) 
  ↓ [Python validation + escaping]
  ↓ scripts/generate.py
LaTeX Files (.tex)
  ↓ [pdflatex compilation]
  ↓
PDF Output (output/generated/)
  ↓ [pdftotext extraction]
  ↓
Tests verify data completeness
```

### Directory Structure

- **`data/`** - YAML source of truth
  - `personal.yaml` - Contact info, taglines for each variant
  - `experience.yaml` - Work history with tags (development, devops, cloud)
  - `skills.yaml` - Tech stack
  - `education.yaml` - Degrees
  - `certifications.yaml` - Professional certs
  - `strengths.yaml` - Key strengths/qualities
  - `cover-letter-template.yaml` - Cover letter structure template

- **`scripts/`** - Python generation and testing
  - `generate.py` - Main generator: YAML → LaTeX (direct string building, no templates)
  - `generate_ats.py` - Generate plain text ATS-friendly versions
  - `test_data_completeness.py` - Verify all YAML data appears in PDFs

- **`templates/`** - LaTeX templates by variant
  - `software-developer/` - Purple theme (creativity, innovation)
  - `devops-engineer/` - Orange theme (energy, collaboration)
  - `cloud-engineer/` - Steel blue theme (trust, professionalism)
  - `altacv-class/` - AltaCV LaTeX document class

- **`output/`** - Generated outputs (not committed)
  - `generated/` - PDF files
  - `ats/` - Plain text ATS versions

- **`.github/workflows/`** - CI/CD automation
  - `cv-build.yml` - Builds all variants on push to main or manual trigger

### Data Model: Tags System

Experience and certifications use **tags** for variant filtering:

- `development` → appears in Software Developer variant
- `devops` → appears in DevOps Engineer variant  
- `cloud` → appears in Cloud Engineer variant
- Multiple tags: entry appears in multiple variants

Example from `experience.yaml`:
```yaml
- title: "Senior DevOps Engineer"
  company: "Tech Corp Inc"
  tags: ["devops", "cloud"]  # This job appears in DevOps and Cloud variants
  achievements: [...]
```

The `generate.py` script reads the variant name from CLI args and filters entries based on matching tags.

## Key Conventions

### YAML Data Requirements

Each YAML file has strict validation in `generate.py`:

- **`personal.yaml`** requires: first_name, last_name, email, phone, location, website, linkedin, github, taglines (dict with keys matching all variants)
- **`experience.yaml`** - Each entry MUST have: title, company, location, start_date, end_date, tags (list), achievements (list)
- All other YAML files: See data/ examples for structure

### LaTeX Escaping

Special characters in YAML are automatically escaped in `generate.py`:
- `&` → `\&`
- `%` → `\%`
- `$` → `\$`
- Backslashes, braces, tildes, etc. - all handled

Don't manually escape in YAML; let the generator handle it.

### Adding New Variants

1. Create template directory: `templates/my-variant/`
2. Add template file: `templates/my-variant/template.tex.j2` (copy from existing variant, modify colors/sections)
3. Add tagline to `data/personal.yaml`: `taglines: { ..., my-variant: "My Variant Title" }`
4. Tag relevant experience/certifications with a new tag (e.g., `my-specialty`)
5. Update `Makefile`: Add to VARIANTS list and `.github/workflows/cv-build.yml` matrix

### Color Psychology by Variant

- **Software Developer (Purple #7C3AED)**: Creativity, innovation, problem-solving
- **DevOps Engineer (Orange #FF6B35)**: Energy, collaboration, developer enablement
- **Cloud Engineer (Steel Blue #4682B4)**: Trust, reliability, professionalism

Edit template colors in `templates/variant/template.tex.j2` (look for `\definecolor` and color macros).

### PDF vs ATS Text Versions

- **Use PDF** when: direct email, networking, portfolios, LinkedIn, after ATS screening
- **Use TXT** when: online application forms, company career portals, anywhere asking to upload/paste

TXT versions are generated by `scripts/generate_ats.py` and contain no formatting—just structured plain text optimized for keyword extraction.

## Testing Strategy

- **`make test`** extracts text from generated PDFs (using pdftotext) and verifies every YAML entry appears in at least one PDF
- Tests validate by **presence** not exact formatting (accounts for LaTeX rendering variations)
- If test fails: Check for special characters, missing tags, or YAML syntax errors
- Common failures: LaTeX compilation errors (usually special characters or missing closing braces in YAML)

## CI/CD Workflow

**Trigger**: Push to main branch OR manual workflow_dispatch

**Steps**:
1. Checkout code
2. Install Python (PyYAML), LaTeX (texlive-debian container), PDF tools (pdftotext)
3. Generate .tex files for each variant
4. Compile .tex → PDF with pdflatex
5. Extract PDF text and run data completeness tests
6. Create GitHub release with PDFs attached

**Failure handling**: All variants build independently (fail-fast: false) so one variant failure doesn't block others.

## Common Modifications

### Change Tagline for a Variant
Edit `data/personal.yaml`: `taglines: { variant-name: "New Title" }`

### Modify Template Colors
Edit `templates/variant/template.tex.j2`:
- Search for `\definecolor` or color hex values
- Update hex codes and recompile

### Add New Skill/Cert/Education Entry
Add to appropriate YAML file with required fields and tags:
```yaml
- name: "Docker"
  tags: ["devops", "cloud"]  # Must match at least one variant's filter
```

### Reorder Sections
Currently the order is baked into `generate.py` functions (personal info, experience, skills, education, certifications, strengths). Modify the function output order if needed.

### Debug LaTeX Compilation Failures
Check `output/generated/variant.log` after failed compilation:
```bash
grep -i error output/generated/*.log
```

Most errors are special character escaping or missing `\endgroup` commands.

## Dependencies

- **Python 3.11+** with PyYAML
- **TeX Live**: pdflatex, texlive-latex-extra, texlive-fonts-extra
- **PDF utilities**: pdftotext, pdfinfo (from poppler-utils)
- **Makefile**: Standard GNU Make

Check [README.md](../README.md) for full setup instructions.
