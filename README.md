# PES-VCS — Version Control System Implementation

This repository contains a complete implementation of a Git-like Version Control System.

##  Final Project Status: COMPLETED

All core phases have been implemented and verified:
- **Phase 1**: Content-addressable object store (SHA-256)
- **Phase 2**: Recursive tree object construction
- **Phase 3**: Staging area (Index) with atomic writes
- **Phase 4**: Commit creation and history walking
- **Phase 5 & 6**: Architectural analysis of branching and GC

---

## Submission Documents

The following documents have been prepared for final submission and upload:

1. **[Git History](git_history.txt)**: Full history of the project development.
2. **[Execution Outputs](test_results.md)**: Captured output of the integration test suite.
3. **[Architecture Map](architecture.md)**: Technical breakdown of the system design.
4. **[Screenshot Guide](screenshot_guide.md)**: Step-by-step instructions for visual verification.
5. **[Project Summary](project_summary.md)**: Final report including Phase 5/6 analysis answers.

---

## Usage

### Building
```bash
make clean && make
```

### Basic Commands
```bash
./pes init                     # Initialize a new repo
./pes add <file>               # Stage a file
./pes status                   # View staging area
./pes commit -m "Message"      # Create a snapshot
./pes log                      # View commit history
```

### Running Tests
```bash
make test_objects && ./test_objects
make test_tree && ./test_tree
./test_sequence.sh
```

---

## Implementation Notes
- **Safety**: Uses atomic renames and `fsync` to prevent repository corruption.
- **Efficiency**: Optimized `index_save` using pointer sorting to handle large staging areas without stack overflow.
- **Portability**: Fixed stack size constraints for compatibility with Darwin/macOS and Linux environments.
