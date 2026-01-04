# PRISM-Zero Engineering Standards
## Zero-Mock Protocol for Sovereign Implementation

---

## 🛡️ **PRISM-ZERO ENGINEERING STANDARD [STRICT MODE]**

**This document establishes mandatory engineering standards for all PRISM-Zero development, ensuring production-grade quality and scientific integrity.**

---

### **1. NO MOCKS / NO SYNTHETIC DATA:**

**Banned Practices:**
- `mock_data()`, `random_f32()`, or placeholder arrays
- Synthetic protein sequences or structures
- Dummy datasets for testing convenience
- Simplified "toy" examples instead of real biological data

**Required Practices:**
- Code must ingest real binary data (from .ptb or io_uring buffers)
- Use authentic datasets: NCBI GenBank, RCSB PDB, UniProt, GISAID
- If data is missing, the code must panic or return a specific `DataMissingError`, not fallback to defaults
- All validation must use cryptographically verified real datasets

**Example - BANNED:**
```rust
// ❌ BANNED: Mock data generation
fn generate_mock_protein() -> Vec<f32> {
    vec![1.0; 1000] // Synthetic coordinates
}
```

**Example - REQUIRED:**
```rust
// ✅ REQUIRED: Real data with integrity validation
pub fn load_verified_pdb(path: &str) -> Result<ProteinStructure, DataIntegrityError> {
    let data = std::fs::read(path)?;
    validate_pdb_integrity(&data)?; // Cryptographic validation
    parse_real_structure(data)
}
```

### **2. NO HARDCODING:**

**Banned Practices:**
- `const PATH = "/tmp/test.pdb";` - Hardcoded paths
- `return 0.5;` - Magic numbers without derivation
- Fixed array sizes based on "example" data
- Embedded configuration constants

**Required Practices:**
- `let path = std::env::var("PRISM_DATA_PATH")?;` or CLI arguments
- `return calculate_entropy(buffer);` - Real computational logic
- Dynamic sizing based on actual data dimensions
- External configuration files or environment variables

**Example - BANNED:**
```rust
// ❌ BANNED: Hardcoded magic numbers
const BINDING_THRESHOLD: f32 = 0.5; // Where did this come from?
const MAX_ATOMS: usize = 10000;     // Arbitrary limit
```

**Example - REQUIRED:**
```rust
// ✅ REQUIRED: Computed or configurable values
pub fn calculate_binding_threshold(energy_profile: &[f32]) -> f32 {
    energy_profile.iter().sum::<f32>() / energy_profile.len() as f32 * 1.5
}

pub struct Config {
    pub max_atoms: usize, // From environment or config file
    pub data_path: PathBuf,
}
```

### **3. PRODUCTION SAFETY:**

**Banned Practices:**
- `unwrap()`, `expect()`, `todo!()`, `unimplemented!()`
- Panic on recoverable errors
- Silent failures or ignored errors
- Unsafe code without thorough safety comments

**Required Practices:**
- Proper `Result<T, E>` propagation with custom Error types
- CUDA: Every CUDA call must be wrapped in a macro that checks `cudaPeekAtLastError`
- Graceful degradation for non-critical failures
- Comprehensive error handling with context

**Example - BANNED:**
```rust
// ❌ BANNED: Panic-prone error handling
let data = file.read().unwrap(); // Will crash on missing file
let result = calculation().expect("This should work"); // Famous last words
```

**Example - REQUIRED:**
```rust
// ✅ REQUIRED: Proper error handling
#[derive(thiserror::Error, Debug)]
pub enum ValidationError {
    #[error("Data integrity violation: {0}")]
    IntegrityViolation(String),
    #[error("I/O error: {0}")]
    IoError(#[from] std::io::Error),
}

pub fn safe_calculation() -> Result<f32, ValidationError> {
    let data = read_verified_data()
        .map_err(|e| ValidationError::IntegrityViolation(format!("Failed to read: {}", e)))?;

    compute_result(data)
        .ok_or_else(|| ValidationError::IntegrityViolation("Computation failed".to_string()))
}
```

### **4. ARCHITECTURAL INTEGRITY:**

**Banned Practices:**
- Python scripts for logic that belongs in Rust/CUDA
- `std::fs::read` for large files (use prism-io async stream)
- Bypassing the sovereign type system
- External dependencies for core algorithms

**Required Practices:**
- Use `prism-io` async streaming for all file operations
- Enforce sovereign data types (no raw `Vec<f32>` for protein data)
- All core physics implemented in Rust/CUDA (zero external ML dependencies)
- Proper module boundaries and abstraction layers

**Example - BANNED:**
```rust
// ❌ BANNED: Raw types that can be mocked
fn process_protein(coords: Vec<f32>) -> f32 {
    // AI can easily pass mock data here
    coords.iter().sum()
}
```

**Example - REQUIRED:**
```rust
// ✅ REQUIRED: Sovereign types that prevent mocking
pub struct SovereignBuffer {
    ptr: *mut c_void,
    len: usize,
    _marker: PhantomData<()>, // Prevents safe construction
}

impl SovereignBuffer {
    pub(crate) unsafe fn new_from_dma(ptr: *mut c_void, len: usize) -> Self {
        Self { ptr, len, _marker: PhantomData }
    }
}

pub fn process_protein(buffer: &SovereignBuffer) -> Result<f32, ProcessingError> {
    // Compiler enforces real data pipeline
    unsafe { compute_from_verified_buffer(buffer.ptr, buffer.len) }
}
```

---

## 🔬 **Three-Pillar Validation Requirements**

### **Data Integrity Enforcement:**

All validation code must implement cryptographic verification:

```rust
use sha2::{Sha256, Digest};

// Real dataset hashes - these MUST match actual data files
const NIPAH_G_GENBANK_SHA256: &str = "a1b2c3d4e5f6789abc...";
const M102_ANTIBODY_PDB_SHA256: &str = "1a2b3c4d5e6f789def...";
const HENDRA_G_UNIPROT_SHA256: &str = "9z8y7x6w5v4u3t2s1r...";

pub fn validate_input_integrity(buffer: &[u8], expected_hash: &str) -> Result<(), DataIntegrityError> {
    let hash = sha256(buffer);
    if hash != expected_hash {
        return Err(DataIntegrityError::IntegrityViolation(
            "Input data is not the authorized biological dataset"
        ));
    }
    Ok(())
}
```

### **Performance Requirements:**

- **Pillar 1 (Escape Mutant)**: <20ms correlation analysis
- **Pillar 2 (Mechanism Re-Discovery)**: <15ms docking validation
- **Pillar 3 (Conserved Cryptic)**: <10ms epitope discovery
- **Total Validation Pipeline**: <50ms end-to-end

### **Success Metrics:**

- **R² > 0.8**: Correlation between PRISM predictions and real mutation hotspots
- **>10x Improvement**: Binding energy improvement vs static methods
- **Novel Targets**: Identification of conserved cryptic sites not in IEDB

---

## ⚡ **Prism-Stream Integration Requirements**

### **Mandatory Data Pipeline:**

All data processing must use the Prism-Stream architecture:

```rust
// ✅ REQUIRED: Prism-Stream data loading
use prism_io::{AsyncPinnedStreamer, HolographicBinaryFormat};

pub async fn load_protein_data(path: &str) -> Result<SovereignBuffer, IoError> {
    let streamer = AsyncPinnedStreamer::new()?;
    let ptb_data = streamer.load_holographic_binary(path).await?;

    // Cryptographic validation before processing
    validate_data_integrity(&ptb_data)?;

    unsafe { Ok(SovereignBuffer::new_from_dma(ptb_data.ptr(), ptb_data.len())) }
}
```

### **Zero-Copy Requirements:**

- All protein data must use memory-mapped `.ptb` format
- GPU transfers via pinned memory only (no host-device copies)
- Warp-drive parsing with CUDA intrinsics for >50x speedup
- io_uring kernel bypass for <1ms data pipeline

---

## 🚨 **Automated Enforcement**

### **CI/CD Pipeline Validation:**

Create `/verify_production.sh` script:

```bash
#!/bin/bash
echo "🔍 Scanning for Non-Production Artifacts..."

# Check for banned Rust patterns
LAZY_RUST=$(grep -rE "todo!|unimplemented!|unwrap\(\)|expect\(\)" crates/ | grep -v "tests/")
if [ ! -z "$LAZY_RUST" ]; then
    echo "❌ FAILED: Found prohibited patterns in production code."
    echo "$LAZY_RUST"
    exit 1
fi

# Check for hardcoded paths
HARDCODED_PATHS=$(grep -rE "\"/tmp/\"|\"/home/\"|\"C:\"" crates/)
if [ ! -z "$HARDCODED_PATHS" ]; then
    echo "❌ FAILED: Found hardcoded absolute paths."
    echo "$HARDCODED_PATHS"
    exit 1
fi

# Check for mock/synthetic data references
MOCKS=$(grep -rE -i "mock|dummy|placeholder|synthetic" crates/ | grep -v "tests/")
if [ ! -z "$MOCKS" ]; then
    echo "❌ FAILED: Found mock/synthetic data references."
    echo "$MOCKS"
    exit 1
fi

# Validate dataset integrity constants
if ! grep -q "SHA256.*[a-f0-9]\{64\}" crates/; then
    echo "❌ FAILED: Missing cryptographic dataset validation."
    exit 1
fi

echo "✅ Production Standards Verified."
```

### **Git Pre-Commit Hook:**

```bash
#!/bin/bash
# .git/hooks/pre-commit
echo "🛡️ PRISM-Zero Production Standards Check..."

if ! ./verify_production.sh; then
    echo "❌ Commit blocked: Production standards violation detected."
    echo "📋 Fix violations and re-commit."
    exit 1
fi

echo "✅ Production standards verified. Proceeding with commit."
```

---

## 📚 **Developer Guidelines**

### **When Working with AI Assistants:**

Always include this header in your prompts:

```markdown
⚠️ PRISM-ZERO ENGINEERING STANDARD [STRICT MODE]

You are working on a production-grade quantum vaccine platform.

MANDATORY REQUIREMENTS:
1. NO MOCKS: Only real biological datasets (GenBank/PDB/UniProt)
2. NO HARDCODING: Use environment variables and configuration
3. NO PANICS: Proper Result<T,E> error handling only
4. USE PRISM-STREAM: All I/O via holographic binary format
5. VALIDATE DATA: SHA-256 integrity verification required
6. SOVEREIGN TYPES: Use SovereignBuffer, not raw Vec<f32>

Any code that violates these standards will be rejected.
```

### **Module Template:**

Use this template for all new modules:

```rust
//! Module: [MODULE_NAME]
//!
//! ⚠️ PRISM-Zero Production Standard Compliance Required
//! See /ENGINEERING_STANDARDS.md for complete requirements
//!
//! Data Sources: [Real datasets only - specify hashes]
//! Performance Target: [Specific timing requirement]
//! Zero-Mock Enforcement: [Describe integrity validation]

use crate::sovereign_types::*;
use anyhow::{Result, Error};
use sha2::{Sha256, Digest};

// Required: Dataset integrity constants
const MODULE_DATASET_SHA256: &str = "your_real_data_hash_here";

#[derive(thiserror::Error, Debug)]
pub enum ModuleError {
    #[error("Data integrity violation: {0}")]
    IntegrityViolation(String),
    #[error("Processing error: {0}")]
    ProcessingError(#[from] ProcessingError),
}

pub fn module_function(input: &SovereignBuffer) -> Result<Output, ModuleError> {
    // Validate input integrity
    validate_buffer_integrity(input)?;

    // Real processing logic - no mocks, no hardcoding
    process_real_data(input)
        .map_err(|e| ModuleError::ProcessingError(e))
}

fn validate_buffer_integrity(buffer: &SovereignBuffer) -> Result<(), ModuleError> {
    // Implementation required - no shortcuts
    unimplemented!("Replace with real cryptographic validation")
}
```

---

## 🎯 **Validation-Specific Requirements**

### **Escape Mutant Analysis:**

```rust
// Required implementation pattern
pub struct EscapeMutantValidator {
    genbank_data: VerifiedGenBankStream,
    hash_validator: Sha256Validator,
}

impl EscapeMutantValidator {
    pub fn new(data_path: &str) -> Result<Self, ValidationError> {
        let genbank_data = VerifiedGenBankStream::load(data_path)?;

        // REQUIRED: Verify this is real NCBI data, not mock
        genbank_data.validate_ncbi_integrity()?;

        Ok(Self {
            genbank_data,
            hash_validator: Sha256Validator::new(),
        })
    }

    pub fn correlate_with_prism(&self, prism_predictions: &SovereignBuffer) -> Result<f32, ValidationError> {
        // Must achieve R² > 0.8 with real mutation data
        let mutation_entropy = self.compute_real_shannon_entropy()?;
        let correlation = self.pearson_correlation(prism_predictions, &mutation_entropy)?;

        if correlation.r_squared < 0.8 {
            return Err(ValidationError::CorrelationTooLow(correlation.r_squared));
        }

        Ok(correlation.r_squared)
    }
}
```

---

## ✅ **Compliance Checklist**

Before any commit, verify:

- [ ] **No Mock Data**: All test data is real biological datasets
- [ ] **Cryptographic Validation**: SHA-256 verification implemented
- [ ] **Error Handling**: All `Result<T,E>` with custom error types
- [ ] **Prism-Stream Integration**: Using holographic binary format
- [ ] **Performance Targets**: Meeting specified timing requirements
- [ ] **Sovereign Types**: No raw `Vec<f32>` for biological data
- [ ] **Zero Hardcoding**: Configuration via environment/files only
- [ ] **Production Safety**: No `unwrap()`, `expect()`, `todo!()`

---

## 📞 **Enforcement Authority**

- **Technical Lead**: Claude Sonnet 4
- **Repository**: https://github.com/Delfictus/Prism4D-bio.git
- **Document**: `/ENGINEERING_STANDARDS.md`
- **Last Updated**: 2026-01-03
- **Review Cycle**: Weekly during active development

**Violations of these standards will result in immediate code review rejection and re-implementation requirements.**

---

*"Production-grade quantum vaccine platforms demand uncompromising engineering standards. Every line of code must meet the highest scientific and technical integrity requirements."*

**🛡️ Zero-Mock Protocol: Enforced ✅**