# AI-Compute — Research + planning workspace

This repo is a **docs-first workspace** for:

- identifying AI niches where customers prefer **dedicated GPU/CPU nodes**
- converting that research into **requirements** (product + ops)
- capturing **customer discovery** scripts/checklists
- outlining a phased path from **concierge → self-serve VMs → templates/jobs → serverless → managed services**

## Quick links (root)

- Roadmap / timeline: [`[R]_roadmap.md`](<[R]_roadmap.md>)
- Provider / vendor questions: [`AllNodes_-_questions.md`](<AllNodes_-_questions.md>)

## Where to start (suggested flow)

1. Pick a niche → [`docs/Top_niches.md`](<docs/Top_niches.md>)
2. Do discovery calls → [`docs/Research_script.md`](<docs/Research_script.md>) and `docs/customer-interview-checklists/`
3. Capture requirements → `docs/niche-requirements-models/`
4. Turn into offers & sales motions → `sales-marketing/agentic-commerce-protocols/`
5. (Optional) Track competitors → `competitors/`

## Docs

### Market & research

- Top niches: [`docs/Top_niches.md`](<docs/Top_niches.md>)
- List of AI niches (often want dedicated nodes):
  [`docs/List_of_AI_niches_where_customers_often_want_dedicated_nodes.md`](<docs/List_of_AI_niches_where_customers_often_want_dedicated_nodes.md>)
- Research script (how to do niche research → ACP contract):
  [`docs/Research_script.md`](<docs/Research_script.md>)

### Product / implementation notes

- App concept + implementation estimates: [`app-concept/`](<app-concept/>)

### Sales & marketing

- Agentic Commerce Protocols (ACPs): [`sales-marketing/agentic-commerce-protocols/`](<sales-marketing/agentic-commerce-protocols/>)

### Competitors

- Competitor notes: [`competitors/`](<competitors/>)

### Customer interview checklists

- [CIC][1][v1] LLM fine-tuning on private corp data:
  [`docs/customer-interview-checklists/[CIC][1][v1]_LLM_fine-tuning_on_private_corp_data.md`](<docs/customer-interview-checklists/[CIC][1][v1]_LLM_fine-tuning_on_private_corp_data.md>)

### Niche Requirements Models (NRM)

All NRMs live in `docs/niche-requirements-models/`.

- [NRM][1][v1] LLM fine-tuning for private corp data:
  [`docs/niche-requirements-models/[NRM][1][v1]_LLM_fine-tuning_for_private_corp_data.md`](<docs/niche-requirements-models/[NRM][1][v1]_LLM_fine-tuning_for_private_corp_data.md>)
- [NRM][2][v1] “RAG factories” (embedding + indexing at scale):
  [`docs/niche-requirements-models/[NRM][2][v1]_RAG_factories_(embedding_+_indexing_at_scale).md`](<docs/niche-requirements-models/[NRM][2][v1]_RAG_factories_(embedding_+_indexing_at_scale).md>)
- [NRM][3][v1] Stable Diffusion / image generation studios:
  [`docs/niche-requirements-models/[NRM][3][v1]_Stable_Diffusion___image_generation_studios.md`](<docs/niche-requirements-models/[NRM][3][v1]_Stable_Diffusion___image_generation_studios.md>)
- [NRM][4][v1] Video generation / video understanding (ad tech, media, VFX):
  [`docs/niche-requirements-models/[NRM][4][v1]_Video_generation___video_understanding_(ad_tech,_media,_VFX).md`](<docs/niche-requirements-models/[NRM][4][v1]_Video_generation___video_understanding_(ad_tech,_media,_VFX).md>)
- [NRM][5][v1] Speech AI (ASR/TTS/voice cloning — legit use):
  [`docs/niche-requirements-models/[NRM][5][v1]_Speech_AI_(ASR_TTS_voice_cloning___legit_use).md`](<docs/niche-requirements-models/[NRM][5][v1]_Speech_AI_(ASR_TTS_voice_cloning___legit_use).md>)
- [NRM][6][v1] Medical imaging (radiology/pathology):
  [`docs/niche-requirements-models/[NRM][6][v1]_Medical_imaging_(radiology_pathology).md`](<docs/niche-requirements-models/[NRM][6][v1]_Medical_imaging_(radiology_pathology).md>)
- [NRM][7][v1] Bioinformatics & protein/genomics modeling:
  [`docs/niche-requirements-models/[NRM][7][v1]_Bioinformatics_&_protein_genomics_modeling.md`](<docs/niche-requirements-models/[NRM][7][v1]_Bioinformatics_&_protein_genomics_modeling.md>)
- [NRM][8][v1] Robotics & embodied AI (sim + policy training):
  [`docs/niche-requirements-models/[NRM][8][v1]_Robotics_&_embodied_AI_(sim_+_policy_training).md`](<docs/niche-requirements-models/[NRM][8][v1]_Robotics_&_embodied_AI_(sim_+_policy_training).md>)
- [NRM][9][v1] Autonomous driving / multi-sensor perception:
  [`docs/niche-requirements-models/[NRM][9][v1]_Autonomous_driving___multi-sensor_perception.md`](<docs/niche-requirements-models/[NRM][9][v1]_Autonomous_driving___multi-sensor_perception.md>)
- [NRM][10][v1] Finance / quant research with ML:
  [`docs/niche-requirements-models/[NRM][10][v1]_Finance___quant_research_with_ML.md`](<docs/niche-requirements-models/[NRM][10][v1]_Finance___quant_research_with_ML.md>)
- [NRM][11][v1] Cybersecurity ML (anomaly, malware classification, log AI):
  [`docs/niche-requirements-models/[NRM][11][v1]_Cybersecurity_ML_(anomaly,_malware_classification,_log_AI).md`](<docs/niche-requirements-models/[NRM][11][v1]_Cybersecurity_ML_(anomaly,_malware_classification,_log_AI).md>)
- [NRM][12][v1] Industrial inspection (factory vision):
  [`docs/niche-requirements-models/[NRM][12][v1]_Industrial_inspection_(factory_vision).md`](<docs/niche-requirements-models/[NRM][12][v1]_Industrial_inspection_(factory_vision).md>)
- [NRM][13][v1] Geospatial / satellite analytics:
  [`docs/niche-requirements-models/[NRM][13][v1]_Geospatial___satellite_analytics_(1).md`](<docs/niche-requirements-models/[NRM][13][v1]_Geospatial___satellite_analytics_(1).md>)
- [NRM][14][v1] Gaming / NPC intelligence / real-time agents:
  [`docs/niche-requirements-models/[NRM][14][v1]_Gaming___NPC_intelligence___real-time_agents.md`](<docs/niche-requirements-models/[NRM][14][v1]_Gaming___NPC_intelligence___real-time_agents.md>)
- [NRM][15][v1] E-commerce personalization / recommender training:
  [`docs/niche-requirements-models/[NRM][15][v1]_E-commerce_personalization___recommender_training.md`](<docs/niche-requirements-models/[NRM][15][v1]_E-commerce_personalization___recommender_training.md>)
- [NRM][16][v1] Enterprise document AI (OCR + extraction + assistants):
  [`docs/niche-requirements-models/[NRM][16][v1]_Enterprise_document_AI_(OCR_+_extraction_+_assistants).md`](<docs/niche-requirements-models/[NRM][16][v1]_Enterprise_document_AI_(OCR_+_extraction_+_assistants).md>)
- [NRM][17][v1] Code models / internal copilots (company codebase):
  [`docs/niche-requirements-models/[NRM][17][v1]_Code_models___internal_copilots_(company_codebase).md`](<docs/niche-requirements-models/[NRM][17][v1]_Code_models___internal_copilots_(company_codebase).md>)
- [NRM][18][v1] Scientific computing / climate / physics surrogates:
  [`docs/niche-requirements-models/[NRM][18][v1]_Scientific_computing___climate___physics_surrogates.md`](<docs/niche-requirements-models/[NRM][18][v1]_Scientific_computing___climate___physics_surrogates.md>)