# DANDI: Distributed Archives for Neurophysiology Data Integration

## Overview

The DANDI platform is supported by the BRAIN Initiative for publishing, sharing, and processing neurophysiology data. The archive accepts cellular neurophysiology data including electrophysiology, optophysiology, and behavioral time-series, and images from immunostaining experiments. The platform is now available for data upload and distribution. The storage of data in the archive is also supported by the Amazon Opendata program.

## Why DANDI?

As an exercise, let's assume you lose all the data in your lab. What would you want from the archive? Our hope is that your answer to this question, the necessary data and metadata that you need, is at least what we should be storing. DANDI provides:

- A cloud-based platform to store, process, and disseminate data. You can use DANDI to collaborate and publish datasets.
- Open access to data to enable secondary uses of data outside the intent of the study.
- Optimize data storage and access through partnerships, compression and accessibility technologies.
- Enables reproducible practices and publications through data standards such as NWB and BIDS.

The platform is not just an endpoint to dump data, it is intended as a living repository that enables collaboration within and across labs.

## Key Resources

### DANDI Archive Home Page
- **URL**: https://about.dandiarchive.org/
- General information about the DANDI archive and its mission

### DANDI Data Portal
- **URL**: https://dandiarchive.org/
- Browse and search for dandisets

### Individual Dandiset Pages
- **Format**: `https://dandiarchive.org/dandiset/XXXXXX`
- Where `XXXXXX` is the dandiset ID
- Example: https://dandiarchive.org/dandiset/001695

## Neurosift Integration

### Viewing Dandisets in Neurosift
- **Format**: `https://neurosift.app/dandiset/XXXXXX`
- Where `XXXXXX` is the dandiset ID
- Example: https://neurosift.app/dandiset/001695

### Viewing Individual NWB File Assets
- **Format**: `https://neurosift.app/nwb?url=https://api.dandiarchive.org/api/assets/{ASSET_ID}/download/&dandisetId={DANDISET_ID}&dandisetVersion={VERSION}`
- Example: https://neurosift.app/nwb?url=https://api.dandiarchive.org/api/assets/52b551c4-2a2f-4a87-82f5-3d804b7cb82a/download/&dandisetId=001695&dandisetVersion=draft

Neurosift provides an interactive viewer for NWB (Neurodata Without Borders) files, allowing you to explore the structure and contents of neurophysiology datasets directly in your browser.
