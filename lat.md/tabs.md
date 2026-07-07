# Tabs

pi-atlas renders six tabs in the dashboard. Each tab is a component that receives a `StatsSummary` and renders its portion of the data.

## Tab Order

`Overview → Languages → Models → Projects → Skills → Usage`

## Overview

KPI cards (total cost, sessions, messages, active days, avg/day, tokens) + ASCII bar chart of daily spend + top-5 ranked lists for languages, models, and skills. Uses `GridRow` for the KPI layout and `BarChart` for the time series.

## Languages

Sorted table of language statistics. Columns: Language, Lines, Cost. Language colors come from `langPalette` in `colorPalette.ts`. Languages are detected from file extensions and editor content in session tool results.

## Models

Sorted table of model usage. Columns: Model, Provider, Calls, Cost. Model names are cleaned by `formatModelName()` (strips provider prefix). Names that overflow the column width trigger `MarqueeText` animation.

## Projects

Sorted table of projects. Columns: Project, Sessions, Cost, Tokens. Project names come from the `session` entry type `project` field in session logs.

## Skills

Sorted table of skill usage. Columns: Skill, Uses, Cost, Tokens. Skills are tracked via explicit `<skill>` tags in user messages and implicit detection of `SKILL.md` file reads in assistant tool calls.

## Usage

Two sections: a `GridRow` of four `StatCard` components (Input Tokens, Output Tokens, Total Tokens, Cost) and a ranked bar list of tool usage (tool name + call count).
