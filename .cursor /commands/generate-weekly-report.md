# Generate Weekly JIRA Accomplishments Report

Generate a comprehensive weekly accomplishments report from JIRA data using MCP tools directly (NO Python scripts, NO Python execution). Generate ONLY the HTML output file `weekly_accomplishments_report.html`. Replicate ALL functionality from `jira_dashboard/weekly_report.py` but implement helper functions inline and process data in-memory.

## CRITICAL: Output Requirements

- **ONLY generate**: `weekly_accomplishments_report.html` file
- **DO NOT create**: Any Python scripts, temporary files, or intermediate processing files
- **DO NOT execute**: Any Python code or scripts
- **Process everything**: In-memory using MCP tools and helper function logic

## Parameters

Parse these from the user's command:
- `--project` or `-p` (required): JIRA project key(s) - single project or comma-separated list (e.g., "PROJ1" or "PROJ1,PROJ2,PROJ3")
- `--days` or `-d` (optional, default: 7): Number of days to look back for analysis
- `--components` or `-c` (optional): Comma-separated components to filter by, in same order as projects (e.g., "Infrastructure,Migration")
- `--main-project` or `-m` (optional): Main project for enhanced bug analysis (overrides MAIN_PROJECT env var)

## Step 1: Verify Environment Variables

Check these are set (use `run_terminal_cmd` with `echo $VAR`):
- `MODEL_API_KEY` (required)
- `SNOWFLAKE_TOKEN` (required)
- `SNOWFLAKE_URL` (required)
- `JIRA_BASE_URL` (required)
- `MAIN_PROJECT` (optional, can be overridden by --main-project)

If any required vars missing, inform user and STOP.

## Step 2: Parse and Normalize Parameters

- Parse projects: handle comma-separated list, normalize to uppercase, remove duplicates
- Parse days: default to 7 if not provided
- Parse components: split by comma, trim whitespace
- Determine effective_main_project: use --main-project if provided, else MAIN_PROJECT env var
- Determine components_provided: true if --components was explicitly provided

## Step 3: Helper Functions to Implement (In-Memory Only)

**IMPORTANT**: Implement these helper functions inline in your processing logic. DO NOT create separate Python files or scripts. Use the logic directly when processing data.

### map_priority(priority_id)
```python
priority_mapping = {
    '10200': 'Normal',
    '3': 'Major',
    '1': 'Blocker',
    '2': 'Critical',
    '10300': 'Undefined',
    '4': 'Minor'
}
return priority_mapping.get(str(priority_id), str(priority_id))
```

### map_status(status_id)
```python
status_mapping = {
    '6': 'Resolved/Closed',
    '10018': 'In Progress',
    '10016': 'New',
    '12422': 'Review',
    '10020': 'To Do',
    '14221': 'Waiting'
}
return status_mapping.get(str(status_id), str(status_id))
```

### map_issue_type(issue_type_id)
```python
issue_type_mapping = {
    '10700': 'Feature',
    '1': 'Bug',
    '16': 'Epic',
    '17': 'Story',
    '3': 'Task'
}
return issue_type_mapping.get(str(issue_type_id), str(issue_type_id))
```

### filter_test_issues(issues)
Remove issues where:
- summary.lower().strip() == 'test'
- summary/description contains "test", "testing", "qa", "debug" (case-insensitive)
- labels contain "test", "qa" (case-insensitive)

Return filtered list.

### get_project_component(project, components_string, projects_list)
**Positional Mapping Logic:**
- Split components_string by comma and trim
- If single component: use for all projects
- Find project index in projects_list
- If index < len(components): return components[index]
- Else: return last component (more projects than components)
- If components_string empty: check PROJECT_COMPONENT_MAPPING env var (JSON), then {PROJECT}_COMPONENT env var, else return project name

### convert_markdown_to_html(text)
1. Convert to string
2. Remove bold formatting: `**text**` → `text`, `__text__` → `text`
3. Convert newlines to `<br>`
4. Convert markdown headers:
   - `### Header` → `<h3>Header</h3>`
   - `## Header` → `<h2>Header</h2>`
   - `# Header` → `<h1>Header</h1>`
   - Handle mixed headers like `## # Header`
5. Convert bullet points: `- item` or `* item` → `<li>item</li>`
6. Convert numbered lists: `1. item` → `<li>item</li>`
7. Wrap consecutive `<li>` in `<ul>` tags
8. Add emojis to accomplishment headers (🏆 ACCOMPLISHED, ⚡ UPGRADES, ✅ COMPLETION, 📋 PLANNING, 🤝 COLLABORATIVE, 🔧 PROCESS)
9. Wrap accomplishment sections in `<div class="accomplishment-category">`
10. Clean up extra `<br>` tags

### add_jira_links_to_html(html_content, project_key, jira_base_url)
1. Get jira_base_url from parameter or JIRA_BASE_URL env var
2. If no URL, return original content
3. Ensure URL ends with `/`
4. Append `browse/` to the base URL if not already present (e.g., `https://your-jira-instance.com/` becomes `https://your-jira-instance.com/browse/`)
5. Pattern: `\b(PROJECT-\d+)\b`
6. For each match:
   - Check if already in `<a>` tag (look backwards/forwards)
   - Check if inside href attribute
   - If not linked, create: `<a href="{jira_base_url}browse/{issue_key}" target="_blank">{issue_key}</a>`
7. Return linked HTML

## Step 4: Process Each Project

For each project in the projects list:

### 4.1: Fetch Recent Issues

**Determine if component filtering needed:**
- `use_components_for_main_issues = main_project AND project == main_project AND components_provided`

**Call MCP tool:** `mcp_jira-mcp-snowflake_list_jira_issues`
- `project`: current project
- `created_days`: days parameter
- `status`: '6' (Closed/Resolved)
- `limit`: 300
- `components`: if `use_components_for_main_issues`, use positional component for this project

**Extract issues** from response: `response['issues']` or `response.get('issues', [])`

### 4.2: Conditional Additional Bug Fetching (Step 1.5)

**Only if:** `main_project AND main_project != project AND components_provided`

**Get project component:** Use `get_project_component(project, components, projects_list)`

**Call MCP tool:** `mcp_jira-mcp-snowflake_list_jira_issues`
- `project`: main_project
- `issue_type`: '1' (Bug)
- `created_days`: days parameter
- `components`: project_component
- `limit`: 300

**Store result** for later merging with bugs list

### 4.3: Conditional Feature Completions (Step 1.6)

**Only if:** `main_project AND main_project != project AND components_provided`

**Get project component:** Use `get_project_component(project, components, projects_list)`

**Call MCP tool:** `mcp_jira-mcp-snowflake_list_jira_issues`
- `project`: main_project
- `issue_type`: '10700' (Feature)
- `status`: '6' (Closed/Resolved)
- `created_days`: days parameter
- `components`: project_component
- `limit`: 300

**Store result** for later merging with analysis data

### 4.4: Fetch Detailed Issue Information (Step 2)

**Batch fetch details** for all issues from Step 4.1:

**Call MCP tool:** `mcp_jira-mcp-snowflake_get_jira_issue_details`
- `issue_keys`: array of all issue keys from Step 4.1

**Enrich issues** with detailed data:
- For each issue from Step 4.1, find matching details in response
- Add: description, comments, labels, links
- Create enriched_issue dict with all fields

### 4.5: Filter Test Issues (Step 2.5)

Apply `filter_test_issues()` to enriched issues list.

### 4.6: Separate Bugs from Non-Bugs (Step 3)

- **Bugs**: `issue_type == '1'`
- **Non-bugs**: all other issue types

**If Step 4.2 ran:** Extend bugs list with additional bugs from main project

### 4.7: Generate Bug Summary (Step 3.5)

**If bugs exist:**

1. **Batch fetch bug details:**
   - Call `mcp_jira-mcp-snowflake_get_jira_issue_details` with all bug keys

2. **Generate qualitative summary** (use LLM, NO numbers):
   - General nature and types of issues
   - Priority level impression (mostly high/low)
   - Overall status impression (mostly open/resolved)
   - Use qualitative terms only: "several", "many", "few", "mostly", "some"
   - 2-3 sentences maximum

### 4.8: Prepare Weekly Accomplishments Data (Step 4)

- **analysis_data = non_bugs.copy()**
- **If Step 4.3 ran:** Extend analysis_data with feature_completions

### 4.9: Generate Weekly Accomplishments Analysis (Step 4.5)

**If analysis_data exists:**

Use LLM to analyze and create markdown report with 6 sections:

1. **## Top 5-8 Things the Team Accomplished This Week**
   - Completed work, resolved issues, delivered features
   - Include issue keys with **ISSUE-KEY** formatting

2. **## Notable Service Upgrades**
   - Infrastructure, performance, monitoring improvements
   - System upgrades, deployment improvements

3. **## Notable Epic or Feature Completion**
   - Completed epics, major features, significant milestones
   - Major deliverables providing business value

4. **## Significant Planning Activities**
   - Strategic planning, design work, architecture decisions
   - Foundational work enabling future development

5. **## Collaborative Efforts with Other Teams**
   - Cross-team initiatives, integration work, partnerships
   - Both internal and external collaborations

6. **## Process Improvements**
   - Workflow optimizations, tool improvements, efficiency gains
   - Automation, standardization, quality improvements

**CRITICAL RULES:**
- Each issue appears in ONLY ONE section (avoid duplicates)
- Use markdown format with `##` headers and `-` bullet lists
- Include issue keys with **ISSUE-KEY** formatting
- Focus on accomplishments, NOT dates
- Priority order if issue fits multiple: Epic/Feature Completion > Service Upgrades > Top Accomplishments > Planning > Collaborative > Process

## Step 5: Generate HTML Report

Create `weekly_accomplishments_report.html` with complete CSS styling.

### HTML Structure:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Weekly Accomplishments Report</title>
    <style>
        /* Full CSS from weekly_report.py lines 777-1148 */
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            line-height: 1.6;
            margin: 0;
            padding: 20px;
            background-color: #f5f5f5;
            color: #333;
        }
        .container {
            max-width: 1200px;
            margin: 0 auto;
            background-color: white;
            padding: 30px;
            border-radius: 8px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        .header {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 30px;
            border-radius: 8px;
            margin: -30px -30px 30px -30px;
            text-align: center;
        }
        .header h1 {
            margin: 0 0 10px 0;
            font-size: 2.5em;
        }
        .header .subtitle {
            font-size: 1.1em;
            opacity: 0.9;
        }
        .meta-info {
            background: #f8f9fa;
            border: 1px solid #e9ecef;
            border-radius: 8px;
            padding: 20px;
            margin: 20px 0;
        }
        .meta-info h3 {
            margin-top: 0;
            color: #495057;
        }
        .project-section {
            margin: 40px 0;
            border: 1px solid #e9ecef;
            border-radius: 8px;
            overflow: hidden;
        }
        .project-header {
            background: #6c757d;
            color: white;
            padding: 15px 25px;
            margin: 0;
            font-size: 1.5em;
            font-weight: bold;
        }
        .project-content {
            padding: 25px;
        }
        .project-content h1, .project-content h2, .project-content h3 {
            color: #495057;
            margin-top: 25px;
            margin-bottom: 15px;
        }
        .project-content h1 {
            color: #2c7be5;
            margin: 0 0 30px 0;
            font-size: 1.7em;
            border-bottom: 3px solid #667eea;
            padding-bottom: 15px;
            display: flex;
            align-items: center;
            gap: 12px;
            font-weight: 600;
        }
        .project-content h1:before {
            content: "🎯";
            font-size: 1.3em;
        }
        .project-content h2 {
            color: #495057;
            margin: 40px 0 25px 0;
            font-size: 1.4em;
            font-weight: 600;
            padding: 20px 25px;
            background: linear-gradient(135deg, #e8f2ff 0%, #f0f7ff 100%);
            border-left: 5px solid #667eea;
            border-radius: 8px;
            display: flex;
            align-items: center;
            gap: 12px;
            box-shadow: 0 3px 10px rgba(102, 126, 234, 0.1);
        }
        .project-content h2:before {
            content: "🔹";
            font-size: 1.2em;
            color: #667eea;
        }
        .project-content h3 {
            color: #6c757d;
            margin: 30px 0 20px 0;
            font-size: 1.2em;
            font-weight: 600;
            padding: 15px 20px;
            background: linear-gradient(135deg, #f8f9fa 0%, #ffffff 100%);
            border-left: 4px solid #6c757d;
            border-radius: 6px;
            display: flex;
            align-items: center;
            gap: 10px;
            box-shadow: 0 2px 8px rgba(108, 117, 125, 0.1);
        }
        .project-content h3:before {
            content: "▶";
            font-size: 0.9em;
            color: #6c757d;
        }
        .project-content ul {
            margin: 25px 0;
            padding: 0;
            list-style: none;
        }
        .project-content li {
            margin: 12px 0;
            padding: 20px 25px;
            background: white;
            border-radius: 12px;
            border-left: 5px solid #667eea;
            line-height: 1.8;
            box-shadow: 0 3px 12px rgba(0, 0, 0, 0.08);
            position: relative;
            transition: all 0.3s ease;
            font-size: 1.05em;
        }
        .project-content li:hover {
            transform: translateX(8px);
            box-shadow: 0 6px 20px rgba(102, 126, 234, 0.2);
        }
        .project-content li:before {
            content: "●";
            color: #667eea;
            font-weight: bold;
            position: absolute;
            left: 8px;
            top: 30px;
            font-size: 12px;
        }
        .project-content strong {
            color: #495057;
            font-weight: 600;
        }
        .project-content p {
            margin: 20px 0;
            line-height: 1.8;
            color: #444;
            font-size: 1.05em;
        }
        .project-content a {
            color: #667eea;
            font-weight: 500;
            text-decoration: none;
            padding: 2px 4px;
            border-radius: 3px;
            background: rgba(102, 126, 234, 0.1);
            transition: all 0.2s ease;
        }
        .project-content a:hover {
            background: rgba(102, 126, 234, 0.2);
            text-decoration: none;
        }
        .accomplishment-category {
            background: linear-gradient(135deg, #f8f9fa 0%, #ffffff 100%);
            border: 1px solid #e9ecef;
            border-radius: 12px;
            padding: 25px;
            margin: 25px 0;
            box-shadow: 0 4px 15px rgba(0, 0, 0, 0.05);
            border-left: 6px solid #667eea;
        }
        .accomplishment-category h2 {
            margin-top: 0;
            margin-bottom: 20px;
            color: #2c7be5;
            font-size: 1.3em;
            font-weight: 600;
            display: flex;
            align-items: center;
            gap: 10px;
        }
        .error-message {
            background: #f8d7da;
            border: 1px solid #f5c6cb;
            color: #721c24;
            padding: 15px;
            border-radius: 5px;
            margin: 15px 0;
        }
        .footer {
            margin-top: 40px;
            padding-top: 20px;
            border-top: 1px solid #e9ecef;
            text-align: center;
            color: #6c757d;
            font-size: 0.9em;
        }
        .jira-link {
            color: #0052cc;
            text-decoration: none;
            font-weight: 500;
        }
        .jira-link:hover {
            text-decoration: underline;
        }
        .bugs-section {
            margin: 30px 0;
            background: #fff9f9;
            border: 1px solid #ffebee;
            border-radius: 8px;
            padding: 20px;
        }
        .bugs-section h3 {
            color: #d32f2f;
            margin-top: 0;
            margin-bottom: 15px;
            font-size: 1.2em;
        }
        .bug-summary {
            background: #f8f9fa;
            border-left: 4px solid #d32f2f;
            border-radius: 4px;
            padding: 15px;
            margin: 15px 0;
            font-style: italic;
            color: #5a5a5a;
            line-height: 1.6;
        }
        .bug-summary p {
            margin: 0;
        }
        .bugs-table {
            width: 100%;
            border-collapse: collapse;
            background: white;
            border-radius: 6px;
            overflow: hidden;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        .bugs-table th {
            background-color: #d32f2f;
            color: white;
            font-weight: 600;
            padding: 12px;
            text-align: left;
        }
        .bugs-table td {
            padding: 10px 12px;
            border-bottom: 1px solid #f0f0f0;
        }
        .bugs-table tr:hover {
            background-color: #fafafa;
        }
        .bugs-table tr:last-child td {
            border-bottom: none;
        }
        .bugs-separator {
            background: #f8f9fa !important;
        }
        .bugs-separator:hover {
            background: #f8f9fa !important;
        }
        .separator-cell {
            padding: 15px 12px !important;
            text-align: center;
            border-top: 2px solid #dee2e6;
            border-bottom: 2px solid #dee2e6;
        }
        .separator-cell .separator-line {
            display: inline-block;
            width: 30%;
            height: 1px;
            background: #6c757d;
            vertical-align: middle;
        }
        .separator-text {
            display: inline-block;
            padding: 0 15px;
            font-weight: 600;
            color: #6c757d;
            font-size: 0.9em;
            vertical-align: middle;
        }
        .resolved-bug {
            opacity: 0.7;
            background: #f8f9fa;
        }
        .resolved-bug:hover {
            opacity: 1;
            background: #e9ecef;
        }
        @media print {
            body {
                background-color: white;
            }
            .container {
                box-shadow: none;
                margin: 0;
                padding: 20px;
            }
            .header {
                background: #667eea !important;
                -webkit-print-color-adjust: exact;
            }
            .project-header {
                background: #6c757d !important;
                -webkit-print-color-adjust: exact;
            }
        }
        @media (max-width: 768px) {
            body {
                padding: 10px;
            }
            .container {
                padding: 20px;
            }
            .header {
                margin: -20px -20px 20px -20px;
                padding: 20px;
            }
            .header h1 {
                font-size: 2em;
            }
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>📊 Weekly Accomplishments Report</h1>
            <div class="subtitle">Team Progress and Achievements Summary</div>
        </div>

        <div class="meta-info">
            <h3>📋 Report Details</h3>
            <p><strong>Projects Analyzed:</strong> {projects_list}</p>
            <p><strong>Analysis Period:</strong> Last {days} days</p>
            <p><strong>Components:</strong> {components or "All"}</p>
            <p><strong>Report Generated:</strong> {current_timestamp}</p>
        </div>

        {project_sections}

        <div class="footer">
            <p>Report generated by JIRA Weekly Accomplishments System | {current_timestamp}</p>
        </div>
    </div>
</body>
</html>
```

### For Each Project Section:

1. **Convert accomplishments markdown to HTML** using `convert_markdown_to_html()`
2. **Add JIRA links** using `add_jira_links_to_html()`
3. **Generate bugs table:**
   - Separate open bugs (status != '6' and not "resolved"/"closed") from resolved bugs (status == '6' or "resolved"/"closed")
   - Show open bugs first
   - Add separator row if both exist
   - Show resolved bugs with `resolved-bug` class
   - Use `map_status()` and `map_priority()` for labels
   - Header: `🐛 Bugs ({open_count} open, {total_count} total)`
   - Show bug summary above table if available

### HTML Generation Rules:

1. Convert markdown accomplishments to HTML
2. Auto-link all JIRA issue keys
3. Separate open bugs from resolved/closed bugs
4. Use mapping functions for human-readable labels
5. Show bug summary above bugs table
6. Display open/total counts in bugs header

## Step 6: Display Summary Statistics

For each project, show:
- Total issues analyzed
- Bugs found (open vs total)
- Non-bug issues
- Feature completions (if applicable)
- Analysis period
- Components filter
- Bug sources (project only or project + main_project)

## Success Criteria

✅ HTML file created: `weekly_accomplishments_report.html`
✅ All requested projects analyzed
✅ Accomplishments organized in 6 sections (no duplicates)
✅ Bugs listed with qualitative summary
✅ Open/resolved bugs separated
✅ JIRA links working
✅ Beautiful styling matches original script
✅ Statistics shown to user

## Key Principles

- **NO Python scripts** - DO NOT create any Python files (.py). Process everything in-memory.
- **NO Python execution** - DO NOT execute Python code. Use MCP tools and native capabilities only.
- **ONLY HTML output** - Generate ONLY `weekly_accomplishments_report.html` file. No intermediate files.
- **Match original output exactly** - HTML should be identical to script-generated version
- **Handle all edge cases** - Component mapping, main project logic, error handling
- **Qualitative summaries only** - No counting in bug summaries (use "several", "many", "few")
- **Avoid duplicates** - Each issue in ONE section only in accomplishments
- **Be systematic and thorough** - Replicate exact behavior of Python script but implement logic inline
