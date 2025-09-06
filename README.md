# Etsy Listing Generator README

## Overview
This Make.com scenario generates 700 unique Etsy listings from a Google Sheet (`Listings_Control`), using predefined templates and category mappings. Listings are created as Drafts (or Active based on `PublishStatus`), with idempotency checks, uniqueness validation, and error handling.

## Setup
1. **Google Sheets**:
   - Create `Listings_Control` with headers: `CategorySlug`, `SubcategorySlug`, `TemplateID`, `VariantSeed`, `PrimaryKeyword`, `SecondaryKeywords`, `Persona`, `Price`, `Quantity`, `Renewal`, `ImagePackID`, `PublishStatus`, `SKU`, `Ready`, `Attr_Subject`, `Attr_FileType`, `Attr_DownloadType`, `ListingID`, `ProcessedAt`, `Status`, `ErrorMessage`, `EtsyURL`, `VariantUsed`.
   - Create `Listings_Index` for idempotency tracking: `IdempotencyKey`, `ListingID`, `CategorySlug`, `Title`, `Tags`.
   - Set `Ready=TRUE` for rows to process.
2. **Environment Variables**:
   - `SPREADSHEET_ID`: Google Sheet ID for `Listings_Control`.
   - `INDEX_SPREADSHEET_ID`: Google Sheet ID for `Listings_Index`.
   - `ETSY_API_KEY`: Etsy API key.
   - `TEMPLATE_URL`: Base URL for template JSON files.
   - `IMAGE_BASE_URL`: Base URL for image packs.
   - `ADMIN_EMAIL`: Email for summary reports.
3. **Files**:
   - Upload `CategoryMap.json` to Make.com storage or a URL accessible by the scenario.
   - Ensure template JSON files are available at `TEMPLATE_URL/{TemplateID}.json`.
   - Ensure image packs are available at `IMAGE_BASE_URL/{ImagePackID}/images.json`.
4. **Make.com**:
   - Import the scenario JSON.
   - Configure connections for Google Sheets, Etsy, and OpenAI.
   - Enable the scenario.

## Running the Scenario
1. Set `Ready=TRUE` for up to 700 rows in `Listings_Control`.
2. Trigger the scenario manually.
3. Monitor progress in Make.com’s execution logs.
4. Check `Listings_Control` for updated `Status`, `ErrorMessage`, `ListingID`, `EtsyURL`, and `ProcessedAt`.
5. Review the summary report emailed to `ADMIN_EMAIL`.

## QA Mode
- Enable QA mode by setting `"qa_mode": true` in the scenario JSON.
- Limits processing to 5 rows for testing.
- Always review Draft listings before setting `PublishStatus=Active`.

## Updating Templates
- Templates are stored at `TEMPLATE_URL/{TemplateID}.json`.
- Format:
  ```json
  {
    "title_templates": ["{primary_keyword} Template for {persona}", "..."],
    "description_blocks": ["Lead summary...", "..."],
    "tag_pools": [["keyword1", "keyword2"], ["..."]],
    "material_options": ["PDF", "Editable File"]
  }
