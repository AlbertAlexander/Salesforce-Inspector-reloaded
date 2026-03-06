# Salesforce Data Management - Albert's Notes

## Problem Statement

Juggling tables for Salesforce sucks. There's a lot of dumping CSVs, manually doing data conditioning, doing joins to existing data sets to avoid duplication, mapping fields, and reuploading when there's errors.

There's some sophisticated tools out there for duplicate management and data conditioning, but they are bulky, bloated, and expensive. Admins and startups need a quick and lean way to handle data. Any apps that are too effective invariably scale into costly packages or freemium products, or they get acquired.

However, there is one extremely good exception: an open source project called Salesforce Inspector.

## Strengths
- Chrome extension, super lightweight and free
- Copy/paste friendly (ctrl-v from csv, sheets, or excel and it just works)
- Error printing and Retry Failed very convenient and default persistent when pulling in new data sets

## Weaknesses
- Can only match/update on Salesforce ID
- Necessitates exporting table of Salesforce records and using vlookup to get IDs
- Import/export separate processes (unlike SOQL)
- Have to remap columns to fields with every new import
- Dependent on shitty Salesforce data formatting requirements
- Salesforce exports data in a different format than it imports, on top of Excel's autoformat bullshit

## Salesforce Data Hostility Matrix

| Field Type | Export format | Excel bullshit | Import format |
|---|---|---|---|
| Date | "1/21/2026" | Autoformatting | YYYY-MM-DD |
| Datetime | "1/5/2026, 4:00 PM" | | YYYY-MM-DDThh:mm:ss.sssZ |
| Name | Full name supported | | Must separate first and last name |
| Related Object | Object.Name (exporting Id often requires utility field) | | Object.Id |
| Amount | Double | Autoformats to currency | Double only (no currency codes or symbols) |
| Currency | Currency code ("USD") | | Currency Code |

## Context

True DB jocks can use SOQL (Salesforce Query Language) to do sophisticated operations, but using it to import sucks. You need a manifest.txt and everything has to be preformatted.

Data Loader is the best option out there, with saveable mapping. But it's a client-side downloadable application that requires its own auth'd session. Inspector is preferred for almost all operations.

When moving data between instances or to/from integrated systems, CSV is the easiest way. For integrated systems, adding a new field mapping or including previously unsynced objects often does not trigger an update (even for polling-based systems), resulting in data discrepancy. There is an operation called "tickling" -- making a trivial change to a huge batch of objects just to trigger an update across all relevant workflows/update-driven integrations.

## Key Insight on Updates vs Inserts

Updates are worse than Inserts. Inserts don't need to map to existing related object fields. Opportunity Updates require an Opportunity ID, an Owner ID, and an Account ID -- strings don't work, IDs only.

## Desired Flow

1. App auths via user's current Salesforce session (already open in browser)
2. User exports CSV / pastes from clipboard
3. App reads CSV/clipboard selection
4. App auto-formats fields into Salesforce-friendly format
5. App infers column mappings
6. If Update or Upsert, app uses the open session to match against existing records and retrieve ID:
   - First on ID
   - Then email
   - Then name
   - If multiple matches found, warn user
   - Allow user to edit inferred match
7. If not updating required related objects, grabs relevant IDs for required import fields (Owner, Account)
8. Links to inferred match show in UI
9. User corrects mappings and validates

## Background

This tool exists because of a single developer's goodwill -- a man who has less time these days and has left it to the community to manage. It is simply amazing that it exists for free.

## Example Pain Sequence

User needs to import Opportunity data from system A to system B. Some are new, some are updates.

1. Open a report in A, add columns for related object IDs, export CSV
2. Convert CSV to XLSX/Sheet for data conditioning
3. Export all opportunities from B into tab 2. Use ID fields from system A to determine which rows are new.
4. VLOOKUP opportunity names or legacy IDs to create Opportunity ID column
5. Identify new Owner IDs from B and map to corresponding names from A
6. Use arcane formulae to convert Date and Datetime fields to correct format
7. Tell Excel you really do want those currency fields to just be raw text
8. Paste into Salesforce Inspector
9. Map all columns to relevant fields
10. Click upload and hope it works
11. Read errors, correct input data
12. Retry Failed
