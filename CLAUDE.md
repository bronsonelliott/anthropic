# CLAUDE.md - AI Assistant Guide

> **Last Updated**: 2025-11-15
> **Purpose**: Comprehensive guide for AI assistants working with this codebase

---

## Table of Contents

1. [Repository Overview](#repository-overview)
2. [Codebase Structure](#codebase-structure)
3. [Architecture & Design Patterns](#architecture--design-patterns)
4. [Development Workflows](#development-workflows)
5. [Code Conventions](#code-conventions)
6. [Testing & Quality](#testing--quality)
7. [Common Tasks](#common-tasks)
8. [Troubleshooting](#troubleshooting)

---

## Repository Overview

### Purpose
This repository contains a Python-based data analysis tool that leverages Anthropic's Claude API to analyze QuickBooks subscription performance data. The system:

- Processes performance metrics across multiple timeframes (WTD, MTD, QTD, YTD)
- Identifies statistical anomalies in subscription data
- Uses Claude to prioritize findings and generate business insights
- Produces automated intelligence briefs for stakeholders

### Tech Stack
- **Language**: Python 3.x
- **Core Dependencies**:
  - `pandas` - Data manipulation and analysis
  - `anthropic` - Claude API client
- **API Integration**: Anthropic Claude API (Haiku/Sonnet models)

### Key Files
```
/home/user/anthropic/
├── GenAI_Anthropic_API.py  # Main analysis script (499 lines)
├── README.md                # Basic project description
├── .gitattributes          # Git configuration
└── CLAUDE.md               # This file
```

---

## Codebase Structure

### File Organization

#### GenAI_Anthropic_API.py
The monolithic main script organized into logical sections:

1. **Configuration Block** (lines 15-52)
   - API credentials and model settings
   - Data file paths
   - Analysis parameters (TOP_N, COMPARISON_TYPES, PIVOT_DIMS)
   - CSV data types for performance optimization

2. **Utility Functions** (lines 56-137)
   - `_pivot_to_scenarios()` - Convert scenario columns to wide format
   - `_aggregate_and_flag()` - Vectorized anomaly detection

3. **API Client Class** (lines 142-204)
   - `ClaudeClient` - Wrapper for Anthropic API with retry logic

4. **Core Analysis** (lines 208-253)
   - `scan_file()` - Multi-dimensional anomaly detection

5. **Reporting** (lines 258-323)
   - `generate_daily_brief()` - Format final intelligence report

6. **Main Execution** (lines 328-499)
   - Orchestrates data loading, analysis, Claude prioritization, and report generation

### Data Flow

```
CSV Files (WTD/MTD/QTD/YTD)
    ↓
Load with explicit dtypes
    ↓
Pivot to scenarios (Actual, Forecast, PY)
    ↓
Multi-level aggregation (Overall → Country → Product → Channel → Combinations)
    ↓
Variance calculation (% and absolute)
    ↓
Filter top/bottom N extremes per timeframe
    ↓
Consolidate all anomalies
    ↓
Claude API prioritization
    ↓
Generate formatted brief
    ↓
Save to file
```

---

## Architecture & Design Patterns

### Design Principles

1. **Performance First**
   - Vectorized operations using pandas (100-500x faster than iterrows)
   - Explicit dtype specification for 25-30% faster CSV reads
   - Single concat operations instead of repeated appends
   - `observed=True` in pivot_table to avoid FutureWarning overhead

2. **Resilience**
   - Exponential backoff retry logic for API calls
   - Comprehensive error handling (FileNotFound, EmptyData, APIError)
   - Graceful degradation (continues on individual file failures)

3. **Modularity**
   - Separate concerns: data processing, API interaction, reporting
   - Helper functions eliminate code duplication
   - Configuration centralized at top of file

### Key Patterns

#### Vectorized Anomaly Detection
```python
# AVOID: Iterating over rows (slow)
for idx, row in df.iterrows():
    variance = (row['Actual'] - row['Forecast']) / row['Forecast']

# PREFER: Vectorized operations (fast)
df['variance_pct'] = ((df['Actual'] - df['Forecast']) / df['Forecast'] * 100).round(1)
```

#### API Retry Pattern
```python
for attempt in range(max_retries):
    try:
        response = api_call()
        return response
    except APIError as e:
        if attempt < max_retries - 1:
            sleep(1.5 * (attempt + 1))  # Exponential backoff
        else:
            return fallback_value
```

#### Categorical Data Handling
```python
# Convert categorical back to string to avoid "new category" errors
for col in pivot.columns:
    if hasattr(pivot[col], 'cat'):
        pivot[col] = pivot[col].astype(str)
```

---

## Development Workflows

### Setup & Configuration

1. **Install Dependencies**
   ```bash
   pip install pandas anthropic
   ```

2. **Configure API Key**
   - Edit line 20 in `GenAI_Anthropic_API.py`
   - Replace `"ANTHROPIC_API_KEY"` with actual key
   - **SECURITY**: Never commit real API keys to version control

3. **Prepare Data Files**
   - Place CSV files in repository root:
     - `gns_wtd.csv`
     - `gns_mtd.csv`
     - `gns_qtd.csv`
     - `gns_ytd.csv`
   - Required columns: `scenario`, `country`, `product`, `channel`, `value`

4. **Run Analysis**
   ```bash
   python GenAI_Anthropic_API.py
   ```

### Configuration Options

#### Model Selection (lines 23-26)
```python
# Choose one:
MODEL_ID = "claude-sonnet-4-5"    # Best quality, slower, more expensive
MODEL_ID = "claude-haiku-4-5"     # Default: Fast, cost-effective
MODEL_ID = "claude-sonnet-4-20250514"  # Specific version
```

#### Analysis Parameters (lines 41-43)
```python
TOP_N = 3  # Increase to flag more anomalies per timeframe
COMPARISON_TYPES = ['Forecast', 'PY']  # Add/remove comparison bases
PIVOT_DIMS = ['country', 'product', 'channel']  # Modify dimensions
```

#### Performance Tuning (lines 28-29, 46-52)
```python
TEMPERATURE = 0.2  # Lower = more deterministic (0.0-1.0)
MAX_TOKENS = 4000  # Increase for longer Claude responses

# Add custom dtypes for new columns
CSV_DTYPES = {
    'scenario': 'category',
    'new_column': 'float32'  # Match data type
}
```

### Git Workflow

**Branch Naming Convention**: `claude/claude-md-{session-id}`

```bash
# Fetch latest
git fetch origin claude/claude-md-mhzt8rhadv78wjj8-01XXRvBwZLBTwpynZvT6Frf1

# Make changes
# ... edit files ...

# Commit
git add .
git commit -m "Descriptive message"

# Push with retry logic (network issues common)
git push -u origin claude/claude-md-mhzt8rhadv78wjj8-01XXRvBwZLBTwpynZvT6Frf1
```

**CRITICAL**: Branch names must start with `claude/` and end with matching session ID. Other patterns will fail with 403 error.

---

## Code Conventions

### Style Guide

1. **Imports**
   - Standard library first
   - Third-party next
   - Local modules last
   - Alphabetical within groups

2. **Naming**
   - `UPPER_CASE` for constants (lines 20-43)
   - `snake_case` for functions and variables
   - `PascalCase` for classes (`ClaudeClient`)
   - Leading underscore for internal helpers (`_pivot_to_scenarios`)

3. **Documentation**
   - Module-level docstring at top (lines 1-7)
   - Function docstrings with Args/Returns
   - Inline comments for complex logic
   - Performance notes where relevant

4. **Error Handling**
   - Specific exceptions before generic
   - Always provide user-friendly error messages
   - Use try-except-finally for cleanup
   - Log errors but don't crash entire pipeline

### Code Patterns to Follow

#### Configuration Management
```python
# GOOD: Centralized configuration at module level
ANTHROPIC_API_KEY = "ANTHROPIC_API_KEY"
MODEL_ID = "claude-haiku-4-5"

# AVOID: Hardcoded values scattered throughout
response = client.ask(model="claude-haiku-4-5")  # Should use MODEL_ID
```

#### DataFrame Operations
```python
# GOOD: Explicit observed=True to avoid warnings
pivot = df.pivot_table(
    index=dims,
    columns='scenario',
    values='value',
    aggfunc='sum',
    observed=True  # Prevents FutureWarning
)

# GOOD: Check for valid data before processing
valid_mask = (df['value'] != 0) & (df['value'].notna()) & (df['actual'] >= 2000)
df_filtered = df[valid_mask].copy()
```

#### String Formatting
```python
# GOOD: f-strings for modern Python
print(f"Flagged {len(extremes)} extremes")

# AVOID: % formatting or .format() unless necessary
print("Flagged %d extremes" % len(extremes))
```

---

## Testing & Quality

### Current State
- **No automated tests**: Manual verification only
- **Quality checks**: Print statements throughout execution
- **Validation**: Spot-check output reports

### Testing Recommendations (for future development)

1. **Unit Tests**
   ```python
   # test_pivot_functions.py
   def test_pivot_to_scenarios():
       sample_df = create_sample_data()
       result = _pivot_to_scenarios(sample_df)
       assert 'Actual' in result.columns
       assert 'Forecast' in result.columns
   ```

2. **Integration Tests**
   - Mock Anthropic API responses
   - Test with sample CSV files
   - Validate report format

3. **Performance Tests**
   - Benchmark large datasets (100K+ rows)
   - Measure vectorization speedup
   - Profile memory usage

### Code Quality Checks

Before committing changes:

1. **Syntax Check**
   ```bash
   python -m py_compile GenAI_Anthropic_API.py
   ```

2. **Linting** (if tools available)
   ```bash
   pylint GenAI_Anthropic_API.py
   flake8 GenAI_Anthropic_API.py
   ```

3. **Manual Verification**
   - Run with sample data
   - Check output file generated
   - Verify Claude response formatted correctly

---

## Common Tasks

### Task 1: Add New Dimension to Analysis

**Goal**: Add `region` dimension alongside country, product, channel

**Steps**:
1. Update PIVOT_DIMS (line 43)
   ```python
   PIVOT_DIMS = ['country', 'product', 'channel', 'region']
   ```

2. Add dtype to CSV_DTYPES (line 48)
   ```python
   CSV_DTYPES = {
       'scenario': 'category',
       'country': 'category',
       'product': 'category',
       'channel': 'category',
       'region': 'category',  # New
       'value': 'float32'
   }
   ```

3. Add aggregation level in scan_file() (after line 251)
   ```python
   _aggregate_and_flag(pivot, 'region', 'Region', timeframe, anomalies)
   _aggregate_and_flag(pivot, ['country', 'region'], 'Country+Region', timeframe, anomalies)
   ```

4. Test with updated CSV files containing `region` column

### Task 2: Change Model or Temperature

**Goal**: Switch to Sonnet for higher quality analysis

**Steps**:
1. Update MODEL_ID (line 24)
   ```python
   MODEL_ID = "claude-sonnet-4-5"
   ```

2. Optional: Adjust temperature for more creative responses (line 27)
   ```python
   TEMPERATURE = 0.5  # Higher = more creative
   ```

3. Optional: Increase max tokens for longer reports (line 28)
   ```python
   MAX_TOKENS = 8000
   ```

4. Run and compare output quality vs. cost

### Task 3: Add Custom Comparison Type

**Goal**: Add "Budget" as a comparison alongside Forecast and PY

**Steps**:
1. Update COMPARISON_TYPES (line 42)
   ```python
   COMPARISON_TYPES = ['Forecast', 'PY', 'Budget']
   ```

2. Ensure CSV files have 'Budget' scenario
   - Verify with: `df['scenario'].unique()`

3. Update logic in _aggregate_and_flag() (lines 109-110)
   ```python
   for comp_name in COMPARISON_TYPES:
       if comp_name == 'Forecast':
           comp_col = 'Forecast'
       elif comp_name == 'PY':
           comp_col = 'PY'
       elif comp_name == 'Budget':
           comp_col = 'Budget'
   ```

4. Test with data containing Budget scenario

### Task 4: Modify Claude Prompt

**Goal**: Change prioritization criteria or output format

**Steps**:
1. Locate prompt in main() function (lines 419-470)

2. Modify instructions:
   ```python
   prompt = f"""You are analyzing New Subscriptions performance.

   NEW INSTRUCTION: Focus only on anomalies >20% variance.
   Provide tactical recommendations, not just hypotheses.

   [Rest of prompt...]
   """
   ```

3. Update expected format section to match new requirements

4. Test and iterate based on Claude's responses

### Task 5: Handle Missing Data Files

**Goal**: Make script resilient to missing files

**Current behavior**: Warns and continues (lines 387-392)

**Enhancement**: Add validation before processing
```python
# Add at start of main()
missing_files = [f for f in FILES.values() if not os.path.exists(f)]
if missing_files:
    print(f"⚠️  Missing files: {missing_files}")
    user_input = input("Continue anyway? (y/n): ")
    if user_input.lower() != 'y':
        return
```

---

## Troubleshooting

### Common Issues

#### 1. API Key Error
```
❌ Setup error: anthropic.AuthenticationError
```
**Solution**: Update ANTHROPIC_API_KEY on line 20 with valid key

#### 2. File Not Found
```
⚠️  File not found: gns_wtd.csv
```
**Solution**:
- Check file paths in FILES dict (lines 34-39)
- Ensure CSV files are in repository root
- Verify filenames match exactly (case-sensitive)

#### 3. Empty DataFrame After Filtering
```
✅ Total anomalies flagged: 0
```
**Solution**:
- Check minimum volume threshold (line 113): `agg['Actual'] >= 2000`
- Verify comparison columns not all zeros
- Lower TOP_N if dataset is small

#### 4. API Rate Limiting
```
API call failed (attempt 1/3): RateLimitError
```
**Solution**:
- Increase sleep time in retry logic (line 200)
- Reduce number of timeframes analyzed
- Switch to Haiku for faster/cheaper processing

#### 5. Memory Issues with Large Datasets
```
MemoryError: Unable to allocate array
```
**Solution**:
- Process timeframes separately instead of concat
- Use chunked reading: `pd.read_csv(filepath, chunksize=10000)`
- Filter data earlier in pipeline
- Use more restrictive dtypes (int16 vs int32)

#### 6. Categorical Column Errors
```
ValueError: Cannot setitem on a Categorical with a new category
```
**Solution**: Already handled in _pivot_to_scenarios() (lines 82-84)
If still occurs, ensure categorical conversion:
```python
df[col] = df[col].astype(str)
```

#### 7. Git Push Fails with 403
```
error: failed to push some refs (403 Forbidden)
```
**Solution**:
- Verify branch name starts with `claude/`
- Ensure branch name ends with correct session ID
- Retry with exponential backoff (network issues)
- Check repository permissions

### Debugging Tips

1. **Enable Verbose Logging**
   ```python
   import logging
   logging.basicConfig(level=logging.DEBUG)
   ```

2. **Inspect DataFrames**
   ```python
   print(df.info())  # Column types and memory
   print(df.describe())  # Statistics
   print(df['column'].value_counts())  # Distribution
   ```

3. **Test API Separately**
   ```python
   bot = ClaudeClient(api_key=ANTHROPIC_API_KEY, model=MODEL_ID)
   response = bot.ask("Test message")
   print(response)
   ```

4. **Validate Data Quality**
   ```python
   # Check for nulls
   print(df.isnull().sum())

   # Check for zeros
   print((df['value'] == 0).sum())

   # Verify scenarios present
   print(df['scenario'].unique())
   ```

---

## AI Assistant Guidelines

### When Assisting with This Codebase

1. **Always Read Before Editing**
   - Use Read tool to view current file state
   - Never make assumptions about line numbers or content

2. **Preserve Performance Optimizations**
   - Don't replace vectorized operations with loops
   - Maintain explicit dtypes
   - Keep observed=True in pivot_table calls

3. **Test Configuration Changes**
   - Verify API key format (don't commit real keys)
   - Check file paths are valid
   - Ensure model IDs are current

4. **Follow Existing Patterns**
   - Use helper functions for repeated logic
   - Add docstrings matching existing format
   - Keep error handling consistent

5. **Consider Downstream Impact**
   - Changing dimensions affects multiple functions
   - Model changes affect cost and latency
   - Prompt changes affect report format

6. **Document Significant Changes**
   - Update this CLAUDE.md file
   - Add inline comments for complex logic
   - Update configuration section if defaults change

### Questions to Ask Before Changing Code

- **Performance**: Will this slow down processing of large datasets?
- **Compatibility**: Does this require CSV schema changes?
- **Cost**: Will this increase API usage or token consumption?
- **Resilience**: How does this handle edge cases (empty data, API failures)?
- **Maintainability**: Is this change obvious to future developers?

---

## Additional Resources

### Anthropic API Documentation
- Main docs: https://docs.anthropic.com/
- API reference: https://docs.anthropic.com/en/api/
- Model comparison: https://docs.anthropic.com/en/docs/about-claude/models

### Python Libraries
- Pandas: https://pandas.pydata.org/docs/
- Anthropic Python SDK: https://github.com/anthropics/anthropic-sdk-python

### Performance Optimization
- Pandas performance tips: https://pandas.pydata.org/docs/user_guide/enhancingperf.html
- Vectorization guide: https://realpython.com/numpy-array-programming/

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-11-15 | Initial CLAUDE.md creation |

---

## Contact & Support

For issues with:
- **Anthropic API**: Check https://docs.anthropic.com/en/api/errors
- **Code questions**: Review this CLAUDE.md and inline documentation
- **Git workflow**: Refer to Development Workflows section above

---

*This guide is maintained for AI assistants working with this codebase. Keep it updated as the project evolves.*
