
# From Software Engineer to AI Builder: A Practical Kickstart 🚀

### Why this article?
If you’re a software engineer and curious about AI, you’ve probably felt lost in jargon: **LLMs, RAG, Agents, Tools, Memory, MCP…**

This guide will demystify those terms and show you how to write a tiny **Proof of Concept (POC)** (C# is used for demonstration).  
We’ll walk through a real healthcare example — *“I have diabetes and swelling in my toes. What should I do?”* — and see how AI pipelines and agentic AI handle it differently.  

By the end, you’ll know how to:
- Understand what happens when a prompt flows through AI components  
- See why agentic approaches matter and their benefits  
- Follow a step-by-step healthcare use case  
- Know what components are needed to build an AI system  
- Understand the role of models vs frameworks  
- Write a tiny POC (both hard-coded pipeline & agentic approaches)  

---

## 1️⃣ What happens when you prompt an AI? 🕵️‍♂️

**Patient’s prompt:**  
👉 “I have diabetes and swelling in my toes. What should I do?”

Here’s the chain of events:  

- **Frontend (UI)** — Collects text input and sends it to the backend.  
- **Backend Orchestration Service** — The conductor; routes the request.  
- **Memory** — Pulls past facts (e.g., patient has diabetes for 10 years).  
- **Embeddings + Vector DB** —  
  - Convert prompt → embedding (vector).  
  - Query vector DB for similar entries (e.g., diabetic-foot guidelines).  
  - Return relevant knowledge chunks.  
- **LLM (Large Language Model)** — Takes prompt + context, generates text or tool requests.  
- **Tools** — Backend-defined functions (e.g., `get_clinical_guidelines`, `get_patient_history`).  
- **MCP (Model Context Protocol)** — The “air-traffic controller” for your AI system.  
  It manages **service discovery, context sharing, and coordination** between LLMs, tools, and memory.  
  It tells the model what tools are available, how to call them safely, and how to access external context such as user data or organizational policies.  
- **Backend** — Combines everything into final output.  
- **Frontend** — Displays safe, summarized advice.  

---

## 2️⃣ Pipeline vs Agentic — Why Agentic Matters 🤔

Imagine you’re training a new nurse.  

- With a **pipeline**, you give them a *script*:  
  - *If the patient has diabetes and their foot is swollen → look up this guideline.*  
  - *If the patient has chest pain → run this test.*  
  - *If the patient has a fever → give this medicine.*  

  This works fine … until the patient asks something you didn’t write in the script. Then the nurse is stuck.  

- With **agentic AI**, instead of a script, you give the nurse a *toolbox*:  
  - They can check medical guidelines, pull lab results, or call a specialist — depending on what the patient says.  
  They’re not following a rigid script; they’re *choosing the right tool in real time.*  

👉 **Why does this matter?**  
Real patients don’t follow scripts. Agentic AI adapts to unexpected inputs without thousands of hard-coded rules.  

💡 **Quick thought experiment:**  
How long would it take to write “if/then” rules for 100 symptoms? Weeks. That’s why agentic AI is the practical choice.  

---

## 3️⃣ Healthcare Example in Action 🏥

**Prompt:**  
👉 “I have diabetes and swelling in my toes. What should I do?”  

When this request flows through the system:  

- **LLM Call** → For a simple general question, the model may answer from its pre-trained knowledge: *“Toe swelling in diabetes could be serious — consult a doctor.”*  
- **RAG (Vector DB)** → Embeddings are compared to a guideline KB (e.g., ADA guidelines on diabetic-foot complications).  
- **Tools (APIs)** → Backend may call domain systems:  
  - **EMR Tool:** `get_recent_lab_reports(patientId)` → returns latest HbA1c, glucose.  
  - **Drug Interaction Tool:** checks whether meds (metformin, insulin) may be related.  
- **Memory** → Patient-specific info (“Diabetes for 10 years,” “reported numbness last month”) personalizes risk assessment.  
- **Final Orchestration (Backend)** → Merge guideline + labs + history + LLM reasoning → deliver safe, contextual recommendation.  

So the same prompt can trigger **LLM**, **RAG**, **Tools**, and **Memory** — each playing a distinct role.  

### 🧩 Who decides what happens?  

- In a **pipeline-based** system, your **backend orchestration service** decides what to do using hard-coded logic — for example:  
  *“If prompt contains ‘diabetes’ and ‘swelling’, fetch diabetic-foot guideline.”*  
  All routing logic lives in your backend.  
- In an **agentic** system, the **LLM itself** — running on **OpenAI’s API** or your **self-hosted model** (vLLM, Ollama, etc.) — decides which tool(s) to call.  
  Your backend only defines the toolbox and executes the tool the model requests.  

👉 The pipeline backend *makes the decisions*; the agentic backend *lets the LLM decide the actions.*  

---

### 💻 Pipeline (hard-coded if/else flow)

```csharp
// Pipeline approach: backend decides using if/else; LLM used for final text generation
if (prompt.Contains("diabetes") && prompt.Contains("swelling")) {
    var guideline = await GetClinicalGuidelines("diabetic foot swelling");

    var advicePrompt = $"Patient: {prompt}\nGuidelines: {guideline}\n"
                     + "Summarize advice in simple language.";
    
    var response = await openAi.Chat.CreateAsync(new ChatCreateRequest {
        Model = "gpt-4o-mini",
        Messages = new[] { new { role = "user", content = advicePrompt } }
    });

    Console.WriteLine(response.Choices[0].Message.Content);
}
else {
    var response = await openAi.Chat.CreateAsync(new ChatCreateRequest {
        Model = "gpt-4o-mini",
        Messages = new[] { new { role = "user", content = prompt } }
    });
    Console.WriteLine(response.Choices[0].Message.Content);
}
```

👉 **Backend decides.** The orchestration service applies rules; the LLM only formats the response.  

---

### 🤖 Agentic (toolbox-driven)

```csharp
// Agentic approach: backend supplies tools; LLM decides which to call
var tools = new[] {
  new { name = "get_clinical_guidelines",
        description = "Fetch treatment guidelines for a condition",
        parameters = new { condition = "string" } },
  new { name = "get_patient_history",
        description = "Retrieve patient’s medical history",
        parameters = new { patientId = "string" } },
  new { name = "drug_interaction_checker",
        description = "Check for conflicts between prescribed drugs",
        parameters = new { drugList = "array" } }
};

var response = await openAi.Chat.CreateAsync(new ChatCreateRequest {
    Model = "gpt-4o-mini",
    Messages = new[] { new { role = "user",
       content = "I have diabetes and swelling in my toes. What should I do?" } },
    Tools = tools
});

// If model calls a tool → backend executes
foreach (var call in response.ToolCalls) {
   var result = call.Name switch {
      "get_clinical_guidelines" => JsonSerialize(await GetClinicalGuidelines("diabetic foot swelling")),
      "get_patient_history"      => JsonSerialize(await GetPatientHistory("JohnDoe")),
      "drug_interaction_checker" => JsonSerialize(await CheckDrugInteractions("JohnDoe")),
      _ => "{}"
   };

   // Feed tool result back into LLM for final synthesis
   var final = await openAi.Chat.CreateAsync(new ChatCreateRequest {
       Model = "gpt-4o-mini",
       Messages = new[] {
          new { role = "user", content = prompt },
          response.Choices[0].Message,
          new { role = "tool", name = call.Name, content = result }
       }
   });
   Console.WriteLine(final.Choices[0].Message.Content);
}
```

👉 **Backend supplies; LLM decides.** Scales easily as you add tools without changing logic.

---

## 4️⃣ What do we need to build an AI system? 🏗️  

Building an AI assistant requires more than just plugging into GPT. Here’s the stack:  

- **UI (Frontend)** → chat box or voice interface  
- **Backend Service** → orchestrates requests, routes to tools, ensures safety  
- **LLM** → reasoning engine (cloud or self-host)  
- **Vector DB (for RAG)** → Chroma, Qdrant, Pinecone, Weaviate  
- **Embeddings Service** → turns text into dense vectors (OpenAI, Hugging Face, local)  
- **Memory Store** → conversation context (Redis, DB)  
- **Tools (APIs/functions)** → custom endpoints for history, guidelines, labs, drug checks  
- **MCP Layer** → coordination, service discovery, compliance, guardrails  

👉 **Your role:** build the backend, define the toolbox, connect RAG + memory, wire UI → backend → LLM.  

---

## 5️⃣ Cloud vs Self-Host: Where Do Toolbox + LLM Live?  

When you build an AI system, you can choose how the LLM runs and connects with your toolbox.  

### (a) Use a paid cloud LLM API (OpenAI, Anthropic, Azure OpenAI)
- Pass the toolbox schema to their hosted LLMs.  
- Cloud LLM decides which tool to call; your backend executes it.  
✅ Simple and fast.  ⚠️ PHI/PII concerns for public cloud.  

### (b) Host Your Own LLM (Private API)
- Deploy inside your VPC, on-prem, or laptop.  
- Toolbox definitions go to your LLM API; model decides tool usage locally.  
✅ More secure.  ⚠️ Needs infra — but commodity hardware can run small models (Mistral 7B, Phi-3).  

### Frameworks + Models for Self-Hosting
- **vLLM** — High-performance (OpenAI-compatible).  Models: LLaMA 3, Mistral, Falcon, Mixtral, Gemma  
- **Ollama** — Lightweight local.  Models: LLaMA 3, Mistral, Phi-3, Gemma  
- **Hugging Face TGI** — Production-grade on-prem/cloud.  Models: Falcon, Mistral, LLaMA 3  
- **LM Studio / LocalAI** — Desktop (OpenAI-compatible, GGUF models)  
- **Enterprise:** Azure OpenAI (VNET), AWS Bedrock, NVIDIA Triton  

### Recommended Setup (Healthcare Hybrid)
- **Cloud + privacy:** Azure OpenAI (VNET) or AWS Bedrock.  
- **Strict on-prem:** vLLM or TGI with LLaMA 3 or Mistral.  
- **Local pilot:** Ollama or LM Studio on laptop.  

👉 **Good news:** Your backend code barely changes — just point the API client to your own endpoint.  

---

## 6️⃣ Wrapping Up 🎯

Using the **diabetes + swelling** example, we’ve seen:

- **Pipelines:** predictable but rigid
- **Agentic AI:** flexible and scalable, needs a well-designed toolbox
- **Building blocks:** UI, backend orchestration, LLM, RAG, embeddings, memory, tools, MCP
- **Toolbox placement:** can integrate with cloud APIs or self-hosted LLMs
- **Models = brains; frameworks = nervous system**

**Takeaway:** You already know APIs and services — AI is just another service that’s probabilistic and needs guardrails.  
Start small:

1. Connect UI → LLM
2. Add RAG (vector DB)
3. Add memory
4. Add tools
5. Flip to agentic

Each step produces a working POC you can demo and extend.

💡 **Next Step in My Journey**  
I’m currently building this **prototype AI system** based on the architecture described here — complete with backend orchestration, RAG, memory, and agentic flows in C#.  
Once ready, I’ll publish the full project on **GitHub** so others can run it locally and extend it for their own use cases.

---

## ✍️ About the Author

**Sudarsan Padmanaban** is a principal engineer passionate about Distributed Systems, cloud architecture, Microservices, Big Data, Messaging, and applied AI.  

He’s currently building a **C#-based AI prototype** that integrates LLMs, RAG, and agentic orchestration — turning theory into a real-world healthcare use case.  

You can follow his work or contribute to the open-source prototype here:  
👉 https://github.com/SudarsanPadmanaban/ai-healthcare-poc-article/tree/main

---

*If this article helped you understand how to bridge software engineering and AI, follow for updates when the full code and implementation guide go live.*
