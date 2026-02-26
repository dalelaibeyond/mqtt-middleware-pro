# Skill: Frontend Log Visualization

## Purpose

Design and implement developer-friendly log visualization UI, optimized for readability, debugging efficiency, and real-time monitoring.

## Capabilities

- Design log layout structure
- Improve log readability
- Apply semantic color coding
- Improve JSON visualization
- Improve scan efficiency
- Reduce cognitive load

## Visualization Principles

### 1. Visual Hierarchy

Use priority order:

Timestamp → Event Type → Topic → Content

Example:

[10:21:33] MESSAGE sensor/temp
{
  "value": 23.5
}

---

### 2. Color Semantics

Use consistent color meaning:

Timestamp → gray  
Info → white  
Success → green  
Warning → yellow  
Error → red  
Topic → cyan  
JSON keys → blue  
JSON values → green  

---

### 3. Layout Rules

Each log entry must be:

- visually separated
- vertically spaced
- aligned
- readable without mental parsing

Use:

- padding
- border-left indicator
- monospace font

---

### 4. JSON Formatting

JSON must be:

- pretty printed
- indented
- colorized
- easy to scan

---

## Output Requirements

When applied, provide:

- improved UI structure
- improved CSS
- improved rendering logic

Focus on readability and developer usability.