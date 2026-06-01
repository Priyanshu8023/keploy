# SMALL_FEATURES_SHORTLIST.md — High-Impact Quick Wins

Based on the feature analysis documents (specifically `QUICK_WINS.md` and `FEATURES.md`), here is a shortlist of the highest-impact, **small features** that can be implemented quickly (ranging from a couple of hours to a few days). 

They are categorized by the specific area they improve:

## 🛠️ Developer Experience (DX) & Usability
These features make Keploy significantly easier and more pleasant to use locally.

*   **`--progress` Flag for Test Execution** *(Effort: 3-4 hours)*
    *   **Improvement:** Adds a live progress bar during `keploy test` (e.g., `[██████░░░] 50% | 3 passed, 1 failed`). Currently, users just see a wall of text.
*   **`--dry-run` Flag for Test Command** *(Effort: 2-3 hours)*
    *   **Improvement:** Allows users to see exactly which tests *would* run (especially useful when combined with tags or filters) without actually executing the app or tests.
*   **`keploy validate` (Schema Validation)** *(Effort: 4-6 hours)*
    *   **Improvement:** Quickly validates all YAML test and mock files for syntax errors, missing fields, or orphaned mocks without having to run a full test suite to find out something is broken.
*   **Watch Mode (`keploy test --watch`)** *(Effort: 2-3 days)*
    *   **Improvement:** Automatically re-runs affected tests when source code files change, massively speeding up the local development inner loop.

## 🚀 CI/CD & Automation
These features make Keploy play much nicer in automated environments like GitHub Actions or GitLab CI.

*   **Markdown Report Format** *(Effort: 3-4 hours)*
    *   **Improvement:** `keploy report --format markdown` generates a clean, GitHub-flavored Markdown table of test results. This makes it trivial to automate commenting test results directly onto Pull Requests.
*   **Test Run Summary in JSON** *(Effort: 2-3 hours)*
    *   **Improvement:** Outputs a structured JSON summary (total, passed, failed, duration) at the end of a run. Makes it incredibly easy for scripts and CI tools to parse the results programmatically.
*   **Test Retention Policies & `keploy clean`** *(Effort: 3-4 hours)*
    *   **Improvement:** Adds config rules (e.g., `maxTestRuns: 10`) and a `clean` command to automatically delete old test reports, preventing CI runners from running out of disk space.

## 🏥 Troubleshooting & Onboarding
These features reduce the burden on maintainers by helping users help themselves.

*   **`keploy doctor` Environment Check** *(Effort: 4-6 hours)*
    *   **Improvement:** A single command that checks if the kernel version supports eBPF, if Docker is running, if required ports are free, and if the config is valid. It outputs a simple ✅/❌ checklist with instructions on how to fix failures.
*   **Keploy Test Badge Generator** *(Effort: 1-2 hours)*
    *   **Improvement:** Generates a shields.io badge URL (e.g., `![Keploy](...21 passed | 3 failed-red)`) that users can embed in their repository's `README.md`.

---

**Recommendation:**
For the most immediate "bang for your buck," start with the **`--progress` Flag**, **Markdown Report Format**, or **`keploy doctor`**. These offer the highest ratio of user-visible impact to implementation time.
