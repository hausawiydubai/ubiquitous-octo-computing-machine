# ubiquitous-octo-computing-machine
AI HAUSAWIY is Qur'an and hadith Ai
"""
Hausawiy Qur’an AI — minimal web API (v0.1)
Owner: Hausawiy World Qur'an Organisation (HWQO)
Authoring assistant: ChatGPT

What this does (today):
- Provides a clean FastAPI endpoint (/ask) for questions about Qur’an & Hadith.
- Uses a single, carefully crafted system prompt to enforce: Arabic text, transliteration, Hausa + English, sources, concise verification path, and voice-over scripts.
- International (auto language detection), but default output includes Hausa & English blocks, plus original Arabic.
- Easy to extend later to WhatsApp Business API or other channels.

What this does NOT do yet:
- It does not ship with a Qur’an/Hadith database. It expects the LLM + future connectors to cite sources.
- It does not include authentication, rate limiting, analytics, or persistence (add later).

How to run locally (dev):
1) Install deps:  pip install fastapi uvicorn pydantic python-dotenv
2) (Optional) If using OpenAI-compatible LLM, also: pip install openai
3) Create a .env file with any provider keys you need, e.g. OPENAI_API_KEY=sk-...
4) Start server:  uvicorn app:app --reload --port 8080
5) Test in browser:  http://127.0.0.1:8080/docs

Later (deployment):
- Put behind HTTPS (e.g., Fly.io, Railway, Render, Azure, etc.).
- Add API auth (bearer token) and per-IP throttling.
- Add WhatsApp webhook handlers when ready.
"""

from __future__ import annotations

import os
from typing import List, Optional, Literal

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field

# -------------------------
# Config & Constants
# -------------------------
APP_NAME = "Hausawiy Qur’an AI"
ORG_NAME = "Hausawiy World Qur'an Organisation"
DEFAULT_VOICE = "calm_male"

# Safety & scope guardrails (keeps the AI on Qur’an/Hadith + general reminders)
SCOPE_RULES = (
    """
You are Hausawiy Qur’an AI, an assistant created by the Hausawiy World Qur'an Organisation (HWQO).

Core scope (allowed & encouraged):
- Qur’an: verses, meanings, memorization guidance, etiquettes of recitation
- Hadith: authentic narrations, references, wisdom
- Seerah & History: prophets' biographies, Sahaba (companions), major scholars and righteous leaders across Islamic history, origins of Islamic sciences/institutions, and formation of the Ummah
- Life advice: reflections on worldly life, ethics, character building, time management, community unity; always grounded in Qur’an & Sunnah with practical steps and relevant examples

Handle carefully / limitations:
- Complex fiqh rulings and personal fatwa → provide neutral information, encourage consulting qualified scholars; do not issue binding rulings.
- Medical, legal, or contemporary political strategy → politely decline and redirect.
- Sectarian disputes → promote unity; cite texts that encourage brotherhood; avoid attacking groups.
- Biographical controversies → present multiple scholarly views briefly with dates and sources; avoid defamation.

Always include: sources, brief verification path, concrete dates when giving history, and keep tone calm, respectful, unifying.
    """
)
    """
You are Hausawiy Qur’an AI, an assistant created by the Hausawiy World Qur'an Organisation (HWQO).
Your primary scope is:
- Qur’an (verses, meanings, memorization guidance, etiquettes of recitation)
- Hadith (authentic narrations, references, wisdom)
- General motivation and reminders grounded in Qur’an and Sunnah

Out of scope / handle carefully:
- Complex fiqh rulings and personal fatwa -> provide neutral information, encourage asking qualified scholars; do not issue binding rulings.
- Medical, legal, or political advice -> politely decline and redirect.
- Sectarian disputes -> promote unity, cite Qur’an/Hadith that encourage brotherhood; avoid attacking groups.

Always include: sources, brief verification path, and keep tone: calm, respectful, unifying.
    """
)

# System prompt (forces structured, bilingual, with Arabic + transliteration)
SYSTEM_PROMPT = f"""
{SCOPE_RULES}

Response policy:
1) Auto-detect the user's language. Open with 1–2 sentences in that language, then ALWAYS include full bilingual blocks (Hausa first, then English). Keep under ~220 words per language unless explicitly asked for more.
2) When quoting Qur’an or Hadith, provide:
   - Arabic text (exact)
   - Transliteration (Latin script, line-aligned)
   - Hausa translation (concise, clear)
   - English translation (concise, clear)
   - Source: (Book/Surah, chapter, verse/hadith number) with common numbering
   - Verification path: short note on where to check (e.g., Quran.com 2:286; sunnah.com Bukhari 1:1)
3) For biographies (prophets, Sahaba, scholars, leaders):
   - Mention full name, kunyah/nisbah if relevant, key dates (hijri + CE if known), place(s), teachers/students
   - 3–6 bullet **highlights** (major contributions, character traits)
   - A short **timeline** of 3–6 dated events
   - Balanced note if there are differing scholarly views
4) For life advice / worldly reflections:
   - Give 2–4 practical steps and 1–2 **examples** relevant to everyday life in clear Hausa & English
   - Ground advice in Qur’an/Hadith with proper sourcing
5) Voice-over scripts: provide two short scripts (Hausa first, then English) suitable for ~20–30s delivery; add 1–2 background suggestions (no instruments).
6) If uncertain about a narration’s authenticity or a historical claim, say so and avoid fabrications; prefer ‘Allah knows best’ and cite cautiously.
7) Keep respectful tone; avoid sectarianism; promote unity of the Ummah.

Final output MUST be valid JSON that conforms to the ResponseModel schema.
"""
"""

# -------------------------
# Data Models
# -------------------------
class Source(BaseModel):
    title: str = Field(..., description="Short title of the source, e.g., 'Qur'an 2:286' or 'Sahih al-Bukhari 1:1'")
    reference: str = Field(..., description="Detailed reference path, e.g., 'Al-Baqarah 2:286; Quran.com/2/286'")

class Voiceover(BaseModel):
    hausa: str
    english: str
    background_suggestions: List[str] = Field(
        default_factory=lambda: [
            "Soft ambient nasheed (no instruments)",
            "Slow pan of Qur’an pages or dawn skyline"
        ]
    )

class QAUnit(BaseModel):
    arabic_text: Optional[str] = None
    transliteration: Optional[str] = None
    translation_hausa: Optional[str] = None
    translation_english: Optional[str] = None
    sources: List[Source] = Field(default_factory=list)
    verification_path: Optional[str] = None
    highlights: List[str] = Field(default_factory=list, description="Key points for biographies or concepts")
    timeline: List[str] = Field(default_factory=list, description="Dated events like '610 CE: First revelation'")
    examples: List[str] = Field(default_factory=list, description="Concrete examples for life advice")

class ResponseModel(BaseModel):
    intro_in_user_language: str = Field(
        ..., description="1–2 sentence intro in the user's detected language"
    )
    category: Optional[str] = Field(
        default=None,
        description="e.g., 'quran', 'hadith', 'biography', 'life_advice'"
    )
    hausa_block: QAUnit
    english_block: QAUnit
    voiceover: Voiceover
    advice: Optional[str] = Field(default=None, description="Short cross-language practical advice summary")
    safety_notes: Optional[str] = Field(
        default=None, description="Used when we must redirect or caution the user"
    )

class AskRequest(BaseModel):
    user_id: Optional[str] = None
    question: str
    prefer_language: Optional[str] = Field(
        default=None,
        description="Optional hint, e.g., 'ha', 'ar', 'en', 'fr' — AI still answers bilingually"
    )

# -------------------------
# LLM Connector (placeholder)
# -------------------------
USE_DUMMY = True  # flip to False when wiring a real LLM

# If you wire a real LLM, use something like this:
# from openai import OpenAI
# client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

def _dummy_llm_compose(req: AskRequest) -> ResponseModel:
    """Return a safe, hard-coded sample showing the expanded JSON shape."""
    user_intro = (
        "Na karɓi tambayarka. Ga amsa a takaice, sannan cikakken bayani a Hausa da English a ƙasa."
        if (req.prefer_language or "").lower().startswith("ha") or "ka" in req.question.lower()
        else "I received your question. Here is a brief answer, followed by full Hausa & English blocks below."
    )

    ha_source = Source(
        title="Qur'an 2:286",
        reference="Al-Baqarah 2:286 – Check: Quran.com/2/286"
    )

    en_source = Source(
        title="Sahih al-Bukhari 1:1",
        reference="The Book of Revelation, #1 – Check: sunnah.com/bukhari:1"
    )

    # Example biography: Umar ibn al-Khattab (r.a.)
    ha_block = QAUnit(
        arabic_text="لَا يُكَلِّفُ ٱللَّهُ نَفْسًا إِلَّا وُسْعَهَا",
        transliteration="Lā yukallifullāhu nafsan illā wus‘ahā",
        translation_hausa="Allah baya ɗora wa rai abin da ya fi ƙarfinsa.",
        translation_english="Allah does not burden a soul beyond its capacity.",
        sources=[ha_source],
        verification_path="Quran.com 2:286",
        highlights=[
            "Umar ibn al-Khattab (r.a.): Khalifa na biyu (13–23 AH / 634–644 CE)",
            "Ya kafa tsarin diwan (alawus/diary) da shari'ar kasuwa a Madina",
            "Ya faɗaɗa adalci da tsari a cikin mulki"
        ],
        timeline=[
            "586 CE: Haihuwa a Makkah",
            "616 CE: Musulunci",
            "634 CE: Ya zama Khalifa",
            "644 CE: Shahada"
        ],
        examples=[
            "Shirya kasuwanci da aminci: tabbatar da ma'auni da nauyi masu adalci",
            "Gudanar da lokaci: sanya masu lura (muhtasib) don kiyaye gaskiya"
        ]
    )

    en_block = QAUnit(
        arabic_text="إِنَّمَا الْأَعْمَالُ بِالنِّيَّاتِ",
        transliteration="Innamā al-a‘mālu bin-niyyāt",
        translation_hausa="Ayyuka suna bisa niyya.",
        translation_english="Actions are judged by intentions.",
        sources=[en_source],
        verification_path="sunnah.com — Bukhari 1",
        highlights=[
            "Imam al-Nawawi: compiled Riyadh as-Salihin; 631–676 AH (1233–1277 CE)",
            "Focus on sincerity (ikhlas) as the root of deeds"
        ],
        timeline=[
            "1233 CE: Birth in Nawa, Syria",
            "1257 CE: Travel to Damascus for study",
            "1277 CE: Passing in Nawa"
        ],
        examples=[
            "Before charity, correct your intention; give quietly",
            "When studying, seek Allah's pleasure first"
        ]
    )

    voice = Voiceover(
        hausa=(
            "(Sauti mai nutsuwa) Ka tuna: Allah baya ɗora maka nauyi fiye da ƙarfinka."
            " Ka tsayu da niyya mai tsarki, ka nemi taimakonSa — zaka dace, in sha Allah."
        ),
        english=(
            "(Calm tone) Remember: Allah never burdens you beyond your capacity."
            " Stand with sincere intention and seek His help — you will find a way."
        ),
    )

    return ResponseModel(
        intro_in_user_language=user_intro,
        category="biography",
        hausa_block=ha_block,
        english_block=en_block,
        voiceover=voice,
        advice="Tsara nufinka, ka nemi taimakon Allah; ka yi aiki cikin adalci. / Purify intention, seek Allah's help, act with justice.",
        safety_notes=None,
    )
        intro_in_user_language=user_intro,
        hausa_block=ha_block,
        english_block=en_block,
        voiceover=voice,
        safety_notes=None,
    )


def compose_response(req: AskRequest) -> ResponseModel:
    if USE_DUMMY:
        return _dummy_llm_compose(req)

    # Example for a real LLM call (pseudocode):
    # messages = [
    #     {"role": "system", "content": SYSTEM_PROMPT},
    #     {"role": "user", "content": req.question},
    # ]
    # resp = client.chat.completions.create(
    #     model="gpt-4o-mini",  # or any preferred model
    #     messages=messages,
    #     temperature=0.3,
    # )
    # content = resp.choices[0].message["content"]
    # parsed = ResponseModel.model_validate_json(content)
    # return parsed

    raise RuntimeError("LLM is not configured. Set USE_DUMMY=True or wire a provider.")

# -------------------------
# FastAPI App
# -------------------------
app = FastAPI(title=APP_NAME, version="0.1", description=f"{APP_NAME} by {ORG_NAME}")


@app.get("/health")
def health():
    return {"status": "ok", "app": APP_NAME, "version": "0.1"}


@app.post("/ask", response_model=ResponseModel)
def ask(req: AskRequest):
    if not req.question or len(req.question.strip()) < 2:
        raise HTTPException(status_code=400, detail="Question is required.")

    try:
        result = compose_response(req)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


# -------------------------
# (Optional) WhatsApp staging hooks (to be implemented later)
# -------------------------
"""
Later we will add:
- /whatsapp/webhook (GET for verification, POST for messages)
- Signature verification (X-Hub-Signature-256)
- Simple router: if message type == text/voice -> forward to /ask; then format back to WhatsApp reply.
"""


{"updates":[{"pattern":"[\s\S]*","multiple":false,"replacement":""""\nAI Hausawiy — Qur’an & Sunnah Web API (Full Version v1.0)\nOwner: Hausawiy World Qur'an Organisation (HWQO)\nAssistant implementer: ChatGPT\n\nWhat this provides (today):\n- Production-ready FastAPI backend with CORS enabled for a web frontend.\n- Single unified /ask endpoint that handles: Qur’an, Hadith, biographies (anbiya’u, sahabbai, scholars), life advice, tajwīd tips, and memorization planning.\n- Structured JSON schema forcing Arabic + transliteration + English + (optional Hausa); with sources and a short verification path.\n- Voice (male-only) TTS endpoint /tts using gTTS as a default (plug-and-play); slots included for Azure/AWS/ElevenLabs male voices.\n- STT placeholder /stt to wire Whisper/Azure later.\n- Memorization planner /learn/plan for custom ḥifẓ schedules (daily/weekly with spaced repetition cues).\n- Static hosting for generated audio under /static/audio/*.\n\nWhat to wire later (optional):\n- Real LLM provider keys (OpenAI, etc.) + real TTS (Azure Neural, Polly, ElevenLabs) for a premium male voice.\n- Persistent DB (PostgreSQL) + Vector DB (FAISS/Pinecone/Weaviate) with Qur’an & Hadith corpora for retrieval-augmented generation (RAG).\n- Admin auth + rate limits + analytics.\n\nRun locally:\n1) pip install fastapi uvicorn pydantic python-dotenv gTTS python-multipart starlette\n2) (Optional, when ready) pip install openai or vendor SDKs\n3) uvicorn app:app --reload --port 8080\n4) Open docs: http://127.0.0.1:8080/docs\n"""\n\nfrom future import annotations\n\nimport os\nimport uuid\nfrom pathlib import Path\nfrom typing import List, Optional\n\nfrom fastapi import FastAPI, HTTPException, UploadFile, File, Form\nfrom fastapi.middleware.cors import CORSMiddleware\nfrom fastapi.staticfiles import StaticFiles\nfrom pydantic import BaseModel, Field\n\ntry:\n    from gtts import gTTS  # default TTS fallback (gender not controllable)\n    HAS_GTTS = True\nexcept Exception:\n    HAS_GTTS = False\n\n# -------------------------\n# Config & Constants\n# -------------------------\nAPP_NAME = "AI Hausawiy"  # ✅ renamed as requested\nORG_NAME = "Hausawiy World Qur'an Organisation"\nDEFAULT_MALE_VOICE = "male_calm_1"  # logical voice id; map to provider later\nAUDIO_DIR = Path("./static/audio")\nAUDIO_DIR.mkdir(parents=True, exist_ok=True)\n\n# -------------------------\n# Safety & scope guardrails\n# -------------------------\nSCOPE_RULES = (\n    """\nYou are AI Hausawiy, the Islamic learning assistant by the Hausawiy World Qur'an Organisation (HWQO).\n\nCore scope (allowed & encouraged):\n- Qur’an: verses, meanings, tajwīd basics (makhārij, ṣifāt, lengthening, stops), memorization guidance & etiquette of recitation\n- Hadith: authentic narrations, references, wisdom (Sahihayn priority; then Sunan; cite clearly)\n- Sīrah & History: prophets’ biographies, Sahaba, major scholars and righteous leaders; key dates (AH/CE)\n- Life advice: reflections grounded in Qur’an & Sunnah; practical steps; good examples\n\nLimitations & handling:\n- Complex fiqh rulings / personal fatwā → provide neutral information, advise consulting qualified scholars; avoid binding rulings.\n- Medical, legal, or political strategy → politely decline and redirect.\n- Sectarian polemics → promote unity; present multiple scholarly views respectfully.\n- Authenticity uncertainty → state uncertainty; avoid fabrications.\n\nAlways include: sources, verification path (quran.com / sunnah.com etc.), and keep a calm, respectful, unifying tone.\n    """\n)\n\nSYSTEM_PROMPT = f"""\n{SCOPE_RULES}\n\nResponse policy:\n1) Auto-detect the user's language. Open briefly (1–2 sentences) in that language, then ALWAYS include bilingual blocks (Arabic text where applicable, Transliteration, English). Optionally add Hausa if context suggests West African audience.\n2) When quoting Qur’an/Hadith, provide:\n   - Arabic text (exact), Transliteration, concise English translation (and Hausa when apt), Sources, Verification path.\n3) For biographies: full name; kunyah/nisbah; key dates (AH/CE); 3–6 highlights; 3–6 timeline entries; balanced note if views differ.\n4) For tajwīd teaching: name the rule; describe makhraj/ṣifah; give 2–3 minimal pairs; a 20–30s practice drill.\n5) For memorization: suggest a daily/weekly plan with review (new/old split), trackable milestones, and dua’.\n6) Provide two short voice-over scripts (Hausa optional) suitable for ~20–30s delivery; background suggestions (no instruments).\n7) Never invent citations; if unknown, say so. JSON must conform to ResponseModel.\n"""\n\n# -------------------------\n# Data Models\n# -------------------------\nclass Source(BaseModel):\n    title: str\n    reference: str\n\nclass Voiceover(BaseModel):\n    hausa: Optional[str] = None\n    english: str\n    background_suggestions: List[str] = Field(\n        default_factory=lambda: [\n            "Soft ambient nasheed (no instruments)",\n            "Slow pan of Qur’an pages or dawn skyline",\n        ]\n    )\n\nclass QAUnit(BaseModel):\n    arabic_text: Optional[str] = None\n    transliteration: Optional[str] = None\n    translation_english: Optional[str] = None\n    translation_hausa: Optional[str] = None\n    sources: List[Source] = Field(default_factory=list)\n    verification_path: Optional[str] = None\n    highlights: List[str] = Field(default_factory=list)\n    timeline: List[str] = Field(default_factory=list)\n    examples: List[str] = Field(default_factory=list)\n    drills: List[str] = Field(default_factory=list, description="Practice lines for tajwid or memorization")\n\nclass ResponseModel(BaseModel):\n    intro_in_user_language: str\n    category: Optional[str] = Field(default=None, description="quran|hadith|biography|life_advice|tajwid|memorization")\n    hausa_block: Optional[QAUnit] = None\n    english_block: QAUnit\n    voiceover: Voiceover\n    advice: Optional[str] = None\n    safety_notes: Optional[str] = None\n\nclass AskRequest(BaseModel):\n    user_id: Optional[str] = None\n    question: str\n    prefer_language: Optional[str] = None  # e.g. 'ar','en','ha','fr' …\n\nclass PlanRequest(BaseModel):\n    goal_surah: Optional[str] = Field(default=None, description="e.g., 'Al-Baqarah' or 'Juz Amma'")\n    days_per_week: int = 5\n    minutes_per_day: int = 30\n    review_ratio: float = 0.5  # portion of time on review\n    base_language: str = "en"\n\nclass PlanResponse(BaseModel):\n    summary: str\n    daily_plan: List[str]\n    reminders: List[str]\n\n# -------------------------\n# LLM connector (dummy for now)\n# -------------------------\nUSE_DUMMY = True\n# If wiring a real LLM, import your SDK and use SYSTEM_PROMPT + messages\n\n\ndef _dummy_llm_compose(req: AskRequest) -> ResponseModel:\n    """Safe illustrative response with tajwid & memorization hints."""\n    q = req.question.lower()\n    intro = (\n        "Assalamu alaikum. I have your question. Full English block is below with Arabic where relevant; Hausa optional."\n    )\n\n    # Sample content branches\n    if any(k in q for k in ["tajwid", "tajweed", "makh", "makhraj"]):\n        category = "tajwid"\n        english_block = QAUnit(\n            arabic_text="قُلْ هُوَ ٱللَّهُ أَحَدٌ",\n            transliteration="Qul huwa Allāhu aḥad",\n            translation_english="Say: He is Allah, the One.",\n            sources=[Source(title="Qur'an 112:1", reference="Al-Ikhlas 112:1 – quran.com/112")],\n            verification_path="quran.com/112",\n            highlights=[\n                "Rule: Qalqalah (echo) on letters ق ط ب ج د when sakin.",\n                "Makhraj: The letter ق from the back of the tongue touching the soft palate.",\n            ],\n            examples=["أَقْرَب — aqrab", "حَقّ — ḥaqq", "يَقُول — yaqūl"],\n            drills=[\n                "Repeat: قَ قِ قُ — focus on back of tongue.",\n                "Read slow: قُلْ هُوَ اللّٰهُ أَحَدٌ (pay attention to ق and shaddah on اللّٰه).",\n            ],\n        )\n        voiceover = Voiceover(\n            english="Practice the Qalqalah gently. Keep the back of the tongue firm for ق. Read slowly and breathe calmly.",\n        )\n        return ResponseModel(\n            intro_in_user_language=intro,\n            category=category,\n

