# Advent of Code Automation Tool

This project provides an automated solution for solving Advent of Code puzzles. It handles:
- Authentication with Advent of Code
- Puzzle discovery and data fetching
- Input parsing and solution generation
- Answer submission and verification

## Environment Setup

1. Ensure Python 3.10+ is installed
2. Create and activate a virtual environment:
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows use: venv\Scripts\activate
   ```
3. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
4. Configure your session cookie in `config.json`

## For Contributors

### Running the Application
1. Activate the virtual environment
2. Run the main script:
   ```bash
   python main.py
   ```

### Testing
To run tests:
```bash
pytest tests/
```

### Development Guidelines
- Follow PEP 8 style guidelines
- Use type hints where appropriate
- Write tests for new features
- Document any new functionality

# **Short Summary and Goal**

We want an end-to-end solution that:
1. **Authenticates** with Advent of Code using a cookie-based approach.
2. **Discovers** which puzzle (day/part) is solvable, potentially assisted by AI to interpret the homepage.
3. **Fetches** and **parses** puzzle content, placing outputs in a configurable directory (default: `tasks`).
4. **Generates or maintains** puzzle solution code (currently optional or placeholder).
5. **Separately Executes** the solution with real input.
6. **Separately Uploads** the puzzle answer to Advent of Code.

The solution must handle nuances:
- If a daily directory already exists, the process will exit rather than overwrite data.
- The second puzzle of the same day is only accessed if the first puzzle is solved (detected via AI or direct checks).

---

# **Main Flow and Concept**

1. **Configuration**  
   - We will have a **configuration file** (e.g., `config.json` or `.ini`) specifying:  
     - Session cookie.  
     - An optional path for storing daily tasks (default = `tasks`).  
   - **Authentication Check**:  
     - Read cookie from config.  
     - Attempt to fetch the Advent of Code homepage to confirm valid credentials.  
     - If not reachable, abort with an error message.

2. **Puzzle Discovery**  
   - If a specific `--day` is provided, we attempt that day first.  
   - Else, an **AI-based** parser will interpret the homepage HTML, looking for the first unsolved day/part.  
   - We also check, via AI or direct parsing, if part 2 is unlocked (i.e., part 1 was solved).  
   - Once a day/part is identified, we verify if a corresponding directory (e.g. `<day>_<part>`) already exists:
     - If **exists**, exit to avoid overwriting.  
     - Otherwise, proceed.

3. **Data Fetching**  
   - Create a subdirectory named `<day>_<part>` within the configured base directory (default `tasks`).  
     - Example: `tasks/6_1`  
   - Retrieve puzzle HTML from `https://adventofcode.com/2024/day/<day>`.  
   - Retrieve puzzle input from `https://adventofcode.com/2024/day/<day>/input`.  
   - Save the input in an `input/` subdirectory (e.g., `tasks/6_1/input/my_input.txt`).

4. **Parsing**  
   - Convert the puzzle HTML to a markdown file named `task.md` (e.g., `tasks/6_1/task.md`).  
   - Identify `<code>` blocks and convert them to fenced code blocks in markdown.  
   - Extract example input/outputs, saving them in `tasks/6_1/input/example-in-<N>.txt` and `tasks/6_1/input/example-out-<N>.txt` respectively.  
   - Maintain a clear structure so further processing (AI solution generation, manual coding, etc.) can easily reference these files.

5. **Solution Generation / Maintenance**  
   - Currently, this may be a placeholder or manual process.  
   - Later, an AI-based approach can read `task.md` (and possibly examples) to generate a Python solver script (e.g., `solve.py`).  
   - If AI-generated, we can store it in the same directory (`tasks/6_1/solve.py`).

6. **Execution of the Real Input**  
   - Separate from the parsing/fetching step, run the solver (e.g., `solve.py`) on `my_input.txt`.  
   - Capture the output, which is typically a single line or short text.  
   - Check logs or store the result in a file (e.g., `result_part1.txt`).

7. **Upload the Result**  
   - After the real input solution is obtained, make an HTTP POST to `https://adventofcode.com/2024/day/<day>/answer`.  
   - Include the answer and part number.  
   - Check the response to determine success or failure.  
   - This step is distinct from simply running the solution (the code can easily chain them but logically they’re separate).

---

# **Suggested Overall Solution Flow**

1. **Initialize and Validate Configuration**  
   - Read session cookie, base directory, and any command-line arguments (like `--day`).  
   - Ensure homepage is reachable with the given cookie.

2. **Discover Puzzle**  
   - If `--day` is set:
     - Use that day, but still confirm whether part 2 is unlocked by analyzing the day’s page.  
   - Otherwise:
     - Fetch homepage HTML → AI identifies the earliest unsolved puzzle day and part.  
   - Verify if `<baseDir>/<day>_<part>` already exists. If yes, exit with a warning.

3. **Create Directory and Fetch Data**  
   - Create `<baseDir>/<day>_<part>/input`.  
   - Fetch puzzle page, puzzle input.  
   - Save them locally for parsing and reference.

4. **Parse to Markdown**  
   - Transform puzzle HTML → `task.md` (retain structure, convert `<code>` → fenced code blocks).  
   - Extract examples → `example-in-*.txt` / `example-out-*.txt`.  
   - Save real input → `my_input.txt`.

5. **Generate or Use Existing Solver**  
   - Optionally use an AI tool or keep a placeholder/manual solver in `solve.py`.  
   - Keep it separate so it can be updated/improved over time.

6. **Test with Example Data**  
   - Run solver on each `example-in-*.txt`; compare to `example-out-*.txt`.  
   - If mismatch, either refine or abort.  
   - If pass, proceed.

7. **Execute on Real Input**  
   - Run solver with `my_input.txt`.  
   - Capture result in memory or a file.

8. **Upload Answer**  
   - POST result to AoC.  
   - Parse success/failure from the response.  
   - Return an appropriate status or message.

---

# **Detailed Phases**

## 1. Configuration and Authentication
1. **Read Config File**:  
   - Example keys: `sessionCookie`, `baseDirectory`.  
   - Fallback to `tasks` if `baseDirectory` not given.
2. **Homepage Connectivity Check**:  
   - Send a GET request to `https://adventofcode.com`.  
   - If status code is 200 (and no “please log in” message), we assume success. Otherwise, stop.

## 2. Puzzle Discovery
1. **Optional CLI Argument** (`--day`):  
   - If provided, skip AI-based discovery.
2. **Homepage Parsing** (AI-based if no `--day`):  
   - Analyze links like `<a aria-label="Day 6"...>`.  
   - Identify which day is unsolved and whether part 2 is open.  
   - Confirm puzzle readiness: check if user already solved it or not.

## 3. Data Fetching
1. **Create Output Directory**:  
   - `<baseDir>/<day>_<part>`.  
   - If it exists, exit to avoid overwriting previous data.
2. **Download Puzzle Page**:  
   - `GET https://adventofcode.com/2024/day/<day>`.
3. **Download Puzzle Input**:  
   - `GET https://adventofcode.com/2024/day/<day>/input`.  
   - Save to `<baseDir>/<day>_<part>/input/my_input.txt`.

## 4. Parsing and Conversion
1. **Convert HTML to `task.md`**:  
   - Extract paragraphs, maintain their structure.  
   - Turn `<code>` blocks into fenced code blocks with appropriate indentation.
2. **Extract Example Inputs/Outputs**:  
   - Often found in code blocks or paragraphs referencing “For example”.  
   - Save each as `input/example-in-<N>.txt` and `input/example-out-<N>.txt`.

## 5. Solution Generation (AI or Manual)
1. **AI Generation** (Future Option):  
   - Provide puzzle statement from `task.md` + examples to an LLM.  
   - Generate a solver script `solve.py`.
2. **Manual**:  
   - User writes or updates the `solve.py` file in the directory.

## 6. Testing with Examples
1. **Run `solve.py`** on each example input file.  
2. **Compare** solver output to the corresponding expected output.  
3. If mismatch, fail or trigger a refinement step.

## 7. Real Input Execution
1. **Run `solve.py`** with `my_input.txt`.  
2. **Capture** output in memory or a file (`result_part1.txt` or similar).

## 8. Result Upload
1. **POST** to `https://adventofcode.com/2024/day/<day>/answer` with the result.  
2. **Check** response HTML or status code for success/failure message.  
3. **Return** the status as boolean or log message.

---

**Potential Improvements/Considerations**  
- Handle partial success/failure if puzzle part 2 is still locked or if the user’s solution is incorrect.  
- Include version control integration (e.g., auto-commit after each day).  
- Allow reruns of the same day by clearing old directories or archiving them rather than forcibly quitting.  
