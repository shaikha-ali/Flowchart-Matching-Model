# Business Grade Matching Pipeline - Flowchart

```mermaid
graph TD
    START([Start]) --> ARGS{Check CLI Arguments}
    
    ARGS -->|--help| HELP["Print help & exit"]
    ARGS -->|--resume-from-step8| RESUME_S8["Load post_step6.pkl<br/>Run Step 8 only<br/>Export Excel"]
    ARGS -->|--resume-export, -r| RESUME_EXP["Load pre_export.pkl<br/>Write Excel only"]
    ARGS -->|--dry-sector-stats| DRY["Load kpmgfile.xlsx<br/>Print sector stats<br/>Exit"]
    ARGS -->|default| FULLPIPE["Full 8-step pipeline"]
    
    HELP --> END([End])
    RESUME_S8 --> END
    RESUME_EXP --> END
    DRY --> END
    
    FULLPIPE --> API_CHECK{"OPENAI_API_KEY<br/>set?"}
    API_CHECK -->|No| ERROR1["❌ Error: API key required"]
    ERROR1 --> END
    API_CHECK -->|Yes| LOAD_FILES["Load kpmgfile.xlsx<br/>Load opportunities file"]
    
    LOAD_FILES --> FIND_COLS["Find/normalize ID columns<br/>Build _company_id, _opp_id"]
    FIND_COLS --> FILL_OPT["Fill optional columns:<br/>industry_field, business_unit_field<br/>hq_field, country_field<br/>website_field"]
    
    FILL_OPT --> VOCAB["Build closed_sector_vocabulary<br/>from company Sector column"]
    VOCAB --> ENRICH_SECTOR["Enrich blank company sectors<br/>using GPT + sector_inference_cache<br/>set sector_was_inferred flag"]
    
    ENRICH_SECTOR --> CART["Create Cartesian product<br/>companies × opportunities<br/>= candidate pairs"]
    
    CART --> STEP1["<b>STEP 1: Preprocessing</b><br/>─────────────────────<br/>Normalize text to lowercase<br/>Remove punctuation<br/>Collapse whitespace"]
    
    STEP1 --> STEP1_OUT["companies._profile_text<br/>companies._sector_norm<br/>opportunities._desc_text<br/>opportunities._sector_norm"]
    
    STEP1_OUT --> STEP2["<b>STEP 2: Sector Ontology</b><br/>─────────────────────<br/>For each unique sector:<br/>→ GPT expands to synonyms,<br/>  sub-domains,<br/>  adjacent domains<br/>→ Cache in sector_ontology_cache.json"]
    
    STEP2 --> STEP2_OUT["expanded_dict:<br/>sector → {synonyms, sub-domains<br/>adjacent_domains}"]
    
    STEP2_OUT --> ENTITY_SETUP["<b>Extract Entities</b><br/>─────────────────────<br/>Load entity_extraction_cache.pkl<br/>Remove cached entries for<br/>companies with inferred sectors<br/>(to re-extract if sector changed)"]
    
    ENTITY_SETUP --> ENTITY_EXT["Parallel GPT entity extraction:<br/>→ Companies: actual_products,<br/>  actual_services, capabilities<br/>→ Opportunities: required_products,<br/>  required_services, required_materials,<br/>  required_capabilities<br/>→ Batch size: EXTRACT_BATCH<br/>→ Workers: PARALLEL_EXTRACT_BATCH<br/>→ Cache: entity_extraction_cache.pkl"]
    
    ENTITY_EXT --> ENTITY_TEXT["Build embedding text:<br/>companies._product_text<br/>opportunities._product_text<br/>opportunities._struct_emb_text<br/>(for structured inputs)"]
    
    ENTITY_TEXT --> STEP4["<b>STEP 4: Embeddings</b><br/>─────────────────────<br/>Embed all text via OpenAI<br/>text-embedding-ada-002<br/>→ 1536-dim vectors"]
    
    STEP4 --> EMB_CHECK{"OpenAI<br/>available?"}
    
    EMB_CHECK -->|Yes| EMB_OPENAI["companies.emb_profile<br/>companies.emb_product<br/>opportunities.emb_desc<br/>opportunities.emb_product<br/>opportunities.emb_struct<br/>Parallel: PARALLEL_EMBED_CHUNKS"]
    
    EMB_CHECK -->|No| EMB_TFIDF["⚠️ Fallback to TF-IDF<br/>Fit vectorizer on all text<br/>Transform each text"]
    
    EMB_OPENAI --> COSINE_SIM["Compute cosine similarity matrices:<br/>sim_profile[ci × oi]<br/>sim_product[ci × oi]<br/>sim_struct_product[ci × oi]"]
    
    EMB_TFIDF --> COSINE_SIM
    
    COSINE_SIM --> TRIAGE["<b>STEP 3+5: Multi-Signal Triage</b><br/>─────────────────────<br/>For EACH company–opportunity pair:"]
    
    TRIAGE --> TRIAGE_LOOP["<b>Compute signals:</b><br/>1️⃣ sector_overlap = sectors_overlap(comp_sector, opp_sector, expanded)?<br/>2️⃣ val_chain_adj = comp_sector in adjacent_value_chain_sectors?<br/>3️⃣ cap_kw_hits = keyword_match_count(capability_keywords, comp_profile+products)?<br/>4️⃣ input_hits = keyword_match_count(required_inputs, comp_products)?<br/>5️⃣ sig_prod = sim_struct_product[ci,oi] score<br/>6️⃣ sig_prof = sim_profile[ci,oi] score"]
    
    TRIAGE_LOOP --> TRIAGE_DECISION{"ANY signal passes<br/>threshold?<br/>─────────────────<br/>sector_overlap == 1<br/>   OR<br/>val_chain_adj == True<br/>   OR<br/>cap_kw_hits ≥ 2<br/>   OR<br/>input_hits ≥ 1<br/>   OR<br/>sig_prod ≥ 0.80<br/>   OR<br/>sig_prof ≥ 0.82"}
    
    TRIAGE_DECISION -->|YES| TRIAGE_PASS["✅ PASS<br/>_send_to_gpt = True<br/>triage_pass = True<br/>Store pass reasons<br/>triage_reason = formatted list<br/>n_to_gpt += 1"]
    
    TRIAGE_DECISION -->|NO| TRIAGE_FAIL["❌ FILTERED<br/>_send_to_gpt = False<br/>triage_pass = False<br/>triage_reason = signal values<br/>n_filtered += 1"]
    
    TRIAGE_PASS --> PAIR_ROW["Add to DataFrame:<br/>companyId, opportunityId<br/>profile_similarity, product_similarity<br/>sector_similarity = max(prof, prod)<br/>ontology_sector_overlap = sector_overlap?<br/>signal_* columns<br/>triage_* columns<br/>ai_score, ai_decision, etc. = stub"]
    
    TRIAGE_FAIL --> PAIR_ROW
    
    PAIR_ROW --> TRIAGE_DONE["All pairs processed<br/>df = DataFrame(rows)<br/>Print: {total_pairs, n_to_gpt, n_filtered}<br/>Print: triage signal hit counts"]
    
    TRIAGE_DONE --> STEP6["<b>STEP 6: GPT Validation</b><br/>─────────────────────<br/>For EACH opportunity in eligible pairs:"]
    
    STEP6 --> STEP6_FILTER["Filter df where _send_to_gpt == True<br/>Group by opportunity._key"]
    
    STEP6_FILTER --> STEP6_LOOP["For each opportunity group:<br/>→ Build opp_context dict<br/>→ Batch candidates (max DECISION_BATCH=14)"]
    
    STEP6_LOOP --> STEP6_GPT["Call gpt_decide_batch():<br/>─ Model: gpt-4.1<br/>─ Input: opportunity_context +<br/>  candidate list<br/>  (name, sector, profile, products,<br/>   extracted entities, match_path,<br/>   ranking_signals)<br/>─ Output: JSON per candidate with<br/>  ai_decision, ai_explanation,<br/>  ai_insight, suggested_plan,<br/>  match_reason"]
    
    STEP6_GPT --> STEP6_DECISION{"GPT response<br/>parsed?"}
    
    STEP6_DECISION -->|Error| STEP6_ERROR["Log error<br/>Default all to ai_decision='No'<br/>Continue"]
    
    STEP6_DECISION -->|Success| STEP6_NORM["Normalize decision:<br/>✅ ai_decision in {Yes, No}<br/>✅ ai_score = 1 if Yes else 0<br/>✅ ai_explanation, ai_insight<br/>✅ suggested_plan (list of 3)<br/>✅ match_reason (list of 3)"]
    
    STEP6_ERROR --> STEP6_UPDATE["Update df rows:<br/>ai_decision, ai_score<br/>ai_explanation, ai_insight<br/>suggested_plan, match_reason"]
    
    STEP6_NORM --> STEP6_UPDATE
    
    STEP6_UPDATE --> STEP6_PARALLEL{Parallel<br/>GPT calls?}
    
    STEP6_PARALLEL -->|Yes| STEP6_THREAD["ThreadPoolExecutor<br/>Workers: PARALLEL_OPP_GPT<br/>Process opportunities concurrently"]
    
    STEP6_PARALLEL -->|No| STEP6_SEQ["Sequential processing<br/>Per opportunity"]
    
    STEP6_THREAD --> STEP6_FILTERED["stamp_sector_gate_filtered():<br/>For rows where _send_to_gpt == False:<br/>→ ai_decision = 'Filtered'<br/>→ ai_score = 0<br/>→ ai_explanation = gate reason<br/>→ match_reason = stub reasons"]
    
    STEP6_SEQ --> STEP6_FILTERED
    
    STEP6_FILTERED --> CHECKPOINT6["Save checkpoint:<br/>df → Output/business_grade_post_step6.pkl<br/>(Resume with --resume-from-step8)"]
    
    CHECKPOINT6 --> STEP8["<b>STEP 8: Ranking & Export</b><br/>─────────────────────<br/>enrich_pair_table_with_source_texts():<br/>→ Merge company_name, company_profile<br/>→ Merge opportunity_name, opportunity_description<br/>→ Fill missing from workbook"]
    
    STEP8 --> DEFAULTS["Ensure all audit columns exist:<br/>sector_was_inferred, sector_inference_source<br/>sector_inference_evidence<br/>signal_*, triage_* columns"]
    
    DEFAULTS --> SECTOR_SIM["Calculate sector_similarity:<br/>sector_similarity = max(profile_similarity,<br/>                        product_similarity)"]
    
    SECTOR_SIM --> FINAL_SCORE["Calculate final_score:<br/>final_score = 0.30×ai_score<br/>             + 0.30×profile_similarity<br/>             + 0.40×product_similarity<br/>Round to 3 decimals"]
    
    FINAL_SCORE --> RANKING["RANKING (per opportunity):<br/>─────────────────────<br/>Sort by opportunityId ASC,<br/>         final_score DESC,<br/>         product_similarity DESC,<br/>         original_row_index ASC<br/>→ Assign rank = 1, 2, 3, ... per opportunity"]
    
    RANKING --> VERIFY_RANK["Sanity check ranking:<br/>✅ final_score monotonic DESC<br/>✅ product_similarity tie-break correct<br/>⚠️ Print any ordering violations"]
    
    VERIFY_RANK --> SANITIZE["Sanitize output:<br/>─ Remove Excel-disallowed ASCII controls<br/>─ JSON-stringify suggested_plan,<br/>  match_reason lists"]
    
    SANITIZE --> BUILD_OUTPUT["Build export DataFrame (31 columns):<br/>id, companyId, opportunityId<br/>company_name, opportunity_name<br/>company_profile, opportunity_description<br/>company_sector, opportunity_sector<br/>sector_similarity, ontology_sector_overlap<br/>profile_similarity, product_similarity<br/>ai_score, ai_decision, final_score<br/>ai_explanation, rank, ai_insight<br/>suggested_plan, match_reason<br/>sector_was_inferred, sector_inference_source<br/>sector_inference_evidence<br/>signal_sector_overlap, signal_value_chain_adjacent<br/>signal_capability_keyword_count<br/>signal_input_overlap_count<br/>signal_product_similarity<br/>signal_profile_similarity<br/>triage_pass, triage_reason"]
    
    BUILD_OUTPUT --> SAVE_PICKLE["Save to pickle:<br/>Output/business_grade_pre_export.pkl<br/>(Recover failed Excel with --resume-export)"]
    
    SAVE_PICKLE --> EXCEL["Write to Excel:<br/>Output/business_grade_company_opportunity_matches.xlsx"]
    
    EXCEL --> FORMAT["Format Excel:<br/>─ Dark header row (white text)<br/>─ Column widths optimized<br/>─ Text wrapping for narratives<br/>─ Freeze panes at A2<br/>─ Color code ai_decision:<br/>  🟢 Yes = light green<br/>  🔴 No = light red<br/>  ⚪ Filtered = light gray"]
    
    FORMAT --> SUMMARY["Print summary:<br/>─ Total rows<br/>─ ai_decision breakdown<br/>─ Yes matches with/without overlap signal<br/>─ final_score percentiles (p10-p99)<br/>─ Sector inference reconciliation"]
    
    SUMMARY --> END
