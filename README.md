{
  "name": "Etsy Listing Generator",
  "flow": [
    {
      "id": 1,
      "module": "trigger:manual",
      "label": "Manual Trigger"
    },
    {
      "id": 2,
      "module": "googlesheets:get_rows",
      "parameters": {
        "spreadsheet_id": "{{SPREADSHEET_ID}}",
        "sheet_name": "Listings_Control",
        "filters": [
          {"column": "Ready", "operator": "equals", "value": "TRUE"},
          {"column": "ProcessedAt", "operator": "is_empty"}
        ],
        "limit": 700
      },
      "label": "Get Eligible Rows"
    },
    {
      "id": 3,
      "module": "iterator",
      "parameters": {
        "array": "{{2.rows}}"
      },
      "label": "Iterate Rows"
    },
    {
      "id": 4,
      "module": "json:parse",
      "parameters": {
        "json": "{{load CategoryMap.json}}"
      },
      "label": "Load Category Map"
    },
    {
      "id": 5,
      "module": "tools:switch",
      "parameters": {
        "value": "{{3.CategorySlug}}",
        "cases": [
          {"case": "career_job_search", "output": "{{4.career_job_search}}"},
          {"case": "wedding_vendor", "output": "{{4.wedding_vendor}}"}
        ],
        "default": {"output": null}
      },
      "label": "Validate Category"
    },
    {
      "id": 6,
      "module": "tools:condition",
      "parameters": {
        "condition": "{{5.output == null || 3.CategorySlug in ['Books, Movies & Music', 'Other']}}"
      },
      "label": "Check Invalid Category",
      "routes": [
        {
          "label": "Invalid Category",
          "flow": [
            {
              "id": 7,
              "module": "googlesheets:update_row",
              "parameters": {
                "spreadsheet_id": "{{SPREADSHEET_ID}}",
                "sheet_name": "Listings_Control",
                "row_id": "{{3.__ROW_ID__}}",
                "values": {
                  "Status": "Error",
                  "ErrorMessage": "Invalid or unmapped category",
                  "ProcessedAt": "{{now}}"
                }
              }
            }
          ]
        },
        {
          "label": "Valid Category",
          "flow": [
            {
              "id": 8,
              "module": "http:get",
              "parameters": {
                "url": "{{TEMPLATE_URL}}/{{3.TemplateID}}.json"
              },
              "label": "Fetch Template"
            },
            {
              "id": 9,
              "module": "tools:sha1",
              "parameters": {
                "text": "{{3.SKU}}:{{3.TemplateID}}:{{3.VariantSeed}}"
              },
              "label": "Generate Idempotency Key"
            },
            {
              "id": 10,
              "module": "googlesheets:search_rows",
              "parameters": {
                "spreadsheet_id": "{{INDEX_SPREADSHEET_ID}}",
                "sheet_name": "Listings_Index",
                "filters": [
                  {"column": "IdempotencyKey", "operator": "equals", "value": "{{9.hash}}"}
                ]
              },
              "label": "Check Idempotency"
            },
            {
              "id": 11,
              "module": "tools:condition",
              "parameters": {
                "condition": "{{10.rows.length > 0}}"
              },
              "label": "Skip if Duplicate",
              "routes": [
                {
                  "label": "Duplicate Found",
                  "flow": [
                    {
                      "id": 12,
                      "module": "googlesheets:update_row",
                      "parameters": {
                        "spreadsheet_id": "{{SPREADSHEET_ID}}",
                        "sheet_name": "Listings_Control",
                        "row_id": "{{3.__ROW_ID__}}",
                        "values": {
                          "Status": "Skipped",
                          "ErrorMessage": "Duplicate idempotency key",
                          "ProcessedAt": "{{now}}"
                        }
                      }
                    }
                  ]
                },
                {
                  "label": "Unique Listing",
                  "flow": [
                    {
                      "id": 13,
                      "module": "openai:chat",
                      "parameters": {
                        "model": "gpt-4o",
                        "system_prompt": "You generate Etsy listing copy that is natural, persuasive, and unique. Never include location words. Title ≤ 140 chars.",
                        "user_prompt": "Create listing content using this JSON:\n{\n  \"template\": {{8.body}},\n  \"primary_keyword\": \"{{3.PrimaryKeyword}}\",\n  \"secondary_keywords\": \"{{3.SecondaryKeywords}}\",\n  \"persona\": \"{{3.Persona}}\",\n  \"variant_seed\": \"{{3.VariantSeed}}\"\n}\nReturn JSON:\n{\n  \"title\": \"…\",\n  \"description_html\": \"…\",\n  \"tags\": [\"…\",\"…\"],\n  \"materials\": [\"…\"],\n  \"attributes\": { \"download_type\": \"digital\" }\n}\nRules:\n- Vary hooks, verbs, and sentence structure with variant_seed.\n- Lead with a 60–90 word summary.\n- Include a bullet list of deliverables with counts.\n- Add use cases, how it works, and 3–5 FAQs.\n- No keyword stuffing, pricing, shipping, or location words."
                      },
                      "label": "Generate Content"
                    },
                    {
                      "id": 14,
                      "module": "code:javascript",
                      "parameters": {
                        "code": "function validateListing(data, categoryMap) {\n  const listing = data[13];\n  const category = categoryMap[5.output];\n  const requiredAttrs = category.attributes_required;\n  const attrs = listing.attributes;\n  for (const attr of requiredAttrs) {\n    if (!attrs[attr]) return { valid: false, error: `Missing attribute: ${attr}` };\n  }\n  if (listing.title.length > 140) return { valid: false, error: 'Title too long' };\n  if (listing.tags.length < 10 || listing.tags.length > 13) return { valid: false, error: 'Invalid tag count' };\n  return { valid: true };\n}\nreturn validateListing({{13}}, {{5.output}});\n"
                      },
                      "label": "Validate Content"
                    },
                    {
                      "id": 15,
                      "module": "tools:condition",
                      "parameters": {
                        "condition": "{{14.valid == false}}"
                      },
                      "label": "Check Validation",
                      "routes": [
                        {
                          "label": "Invalid Content",
                          "flow": [
                            {
                              "id": 16,
                              "module": "googlesheets:update_row",
                              "parameters": {
                                "spreadsheet_id": "{{SPREADSHEET_ID}}",
                                "sheet_name": "Listings_Control",
                                "row_id": "{{3.__ROW_ID__}}",
                                "values": {
                                  "Status": "Error",
                                  "ErrorMessage": "{{14.error}}",
                                  "ProcessedAt": "{{now}}"
                                }
                              }
                            }
                          ]
                        },
                        {
                          "label": "Valid Content",
                          "flow": [
                            {
                              "id": 17,
                              "module": "code:javascript",
                              "parameters": {
                                "code": "function checkUniqueness(title, desc, tags, category, index) {\n  const titleTokens = title.toLowerCase().split(' ');\n  const descStart = desc.slice(0, 400);\n  for (const row of index) {\n    if (row.CategorySlug !== category) continue;\n    const overlap = titleTokens.filter(t => row.Title.includes(t)).length / titleTokens.length;\n    if (overlap > 0.5) return { unique: false, reason: 'Title overlap' };\n    const tagOverlap = tags.filter(t => row.Tags.includes(t)).length;\n    if (tagOverlap > 6) return { unique: false, reason: 'Tag overlap' };\n  }\n  return { unique: true };\n}\nreturn checkUniqueness({{13.title}}, {{13.description_html}}, {{13.tags}}, {{3.CategorySlug}}, {{10.rows}});\n"
                              },
                              "label": "Uniqueness Check"
                            },
                            {
                              "id": 18,
                              "module": "tools:condition",
                              "parameters": {
                                "condition": "{{17.unique == false}}"
                              },
                              "label": "Check Uniqueness",
                              "routes": [
                                {
                                  "label": "Not Unique",
                                  "flow": [
                                    {
                                      "id": 19,
                                      "module": "googlesheets:update_row",
                                      "parameters": {
                                        "spreadsheet_id": "{{SPREADSHEET_ID}}",
                                        "sheet_name": "Listings_Control",
                                        "row_id": "{{3.__ROW_ID__}}",
                                        "values": {
                                          "Status": "Error",
                                          "ErrorMessage": "{{17.reason}}",
                                          "ProcessedAt": "{{now}}"
                                        }
                                      }
                                    }
                                  ]
                                },
                                {
                                  "label": "Unique Listing",
                                  "flow": [
                                    {
                                      "id": 20,
                                      "module": "http:get",
                                      "parameters": {
                                        "url": "{{IMAGE_BASE_URL}}/{{3.ImagePackID}}/images.json"
                                      },
                                      "label": "Fetch Images"
                                    },
                                    {
                                      "id": 21,
                                      "module": "etsy:create_listing",
                                      "parameters": {
                                        "endpoint": "listings",
                                        "method": "POST",
                                        "payload": {
                                          "taxonomy_id": "{{5.output.taxonomy_id}}",
                                          "title": "{{13.title}}",
                                          "description": "{{13.description_html}}",
                                          "tags": "{{13.tags}}",
                                          "materials": "{{13.materials}}",
                                          "attributes": "{{merge(5.output.attributes_defaults, 13.attributes, { subject: 3.Attr_Subject || 5.output.attributes_defaults.subject, file_type: 3.Attr_FileType || 5.output.attributes_defaults.file_type, download_type: 3.Attr_DownloadType || 5.output.attributes_defaults.download_type })}}",
                                          "images": "{{20.body.images.slice(0, 5)}}",
                                          "price": "{{3.Price}}",
                                          "quantity": "{{3.Quantity}}",
                                          "state": "{{3.PublishStatus}}",
                                          "sku": "{{3.SKU}}"
                                        },
                                        "headers": {
                                          "x-api-key": "{{ETSY_API_KEY}}"
                                        },
                                        "retry": {
                                          "attempts": 3,
                                          "delay": "exponential",
                                          "codes": [429, 500, 502, 503]
                                        },
                                        "delay_ms": "{{random(1000, 2000)}}"
                                      },
                                      "label": "Create Etsy Listing"
                                    },
                                    {
                                      "id": 22,
                                      "module": "googlesheets:update_row",
                                      "parameters": {
                                        "spreadsheet_id": "{{SPREADSHEET_ID}}",
                                        "sheet_name": "Listings_Control",
                                        "row_id": "{{3.__ROW_ID__}}",
                                        "values": {
                                          "ListingID": "{{21.listing_id}}",
                                          "EtsyURL": "{{21.url}}",
                                          "ProcessedAt": "{{now}}",
                                          "VariantUsed": "{{3.VariantSeed}}",
                                          "Status": "Created",
                                          "ErrorMessage": ""
                                        }
                                      },
                                      "label": "Update Listing Control"
                                    },
                                    {
                                      "id": 23,
                                      "module": "googlesheets:add_row",
                                      "parameters": {
                                        "spreadsheet_id": "{{INDEX_SPREADSHEET_ID}}",
                                        "sheet_name": "Listings_Index",
                                        "values": {
                                          "IdempotencyKey": "{{9.hash}}",
                                          "ListingID": "{{21.listing_id}}",
                                          "CategorySlug": "{{3.CategorySlug}}",
                                          "Title": "{{13.title}}",
                                          "Tags": "{{13.tags.join(',')}}"
                                        }
                                      },
                                      "label": "Update Listings Index"
                                    }
                                  ]
                                }
                              ]
                            }
                          ]
                        }
                      ]
                    }
                  ]
                }
              ]
            }
          ]
        }
      ]
    },
    {
      "id": 24,
      "module": "aggregator",
      "parameters": {
        "target_module": 3,
        "fields": [
          {"name": "Created", "value": "{{if(21.listing_id, 1, 0)}}"},
          {"name": "Collisions", "value": "{{if(17.unique == false, 1, 0)}}"},
          {"name": "Errors", "value": "{{if(14.valid == false || 5.output == null || 3.CategorySlug in ['Books, Movies & Music', 'Other'], 1, 0)}}"},
          {"name": "ErrorMessage", "value": "{{14.error || 17.reason || 'Invalid category'}}"}
        ]
      },
      "label": "Aggregate Results"
    },
    {
      "id": 25,
      "module": "tools:compose_string",
      "parameters": {
        "template": "Summary Report:\nTotal Listings Processed: {{24.count}}\nCreated: {{sum(24.Created)}}\nCollisions: {{sum(24.Collisions)}}\nErrors: {{sum(24.Errors)}}\nTop Errors: {{groupBy(24.ErrorMessage, 'count').slice(0, 3)}}"
      },
      "label": "Generate Summary Report"
    },
    {
      "id": 26,
      "module": "email:send",
      "parameters": {
        "to": "{{ADMIN_EMAIL}}",
        "subject": "Etsy Listing Run Summary",
        "body": "{{25.result}}"
      },
      "label": "Send Summary Report"
    }
  ],
  "settings": {
    "qa_mode": false,
    "qa_limit": 5
  }
}
