# How Bedrock Guardrails Work: Processing Flow & Logic

## 🔄 Guardrail Processing Flow

### **Sequential Processing with Early Termination on BLOCK**

Bedrock guardrails process content sequentially and **STOP IMMEDIATELY** when any policy blocks:

```mermaid
flowchart TD
    Input[User Input] --> ContentFilter[Content Policy Filters]
    ContentFilter -->|BLOCK| Stop1[❌ IMMEDIATE BLOCK]
    ContentFilter -->|PASS| TopicPolicy[Topic Policy]
    TopicPolicy -->|BLOCK| Stop2[❌ IMMEDIATE BLOCK]
    TopicPolicy -->|PASS| WordPolicy[Word Policy]
    WordPolicy -->|BLOCK| Stop3[❌ IMMEDIATE BLOCK]
    WordPolicy -->|PASS| SensitiveInfo[Sensitive Information Policy]
    SensitiveInfo -->|BLOCK| Stop4[❌ IMMEDIATE BLOCK]
    SensitiveInfo -->|PASS/ANONYMIZE| Allow[✅ ALLOW]
    
    style ContentFilter fill:#FF9800,stroke:#333,stroke-width:2px,color:#fff
    style TopicPolicy fill:#4CAF50,stroke:#333,stroke-width:2px,color:#fff
    style WordPolicy fill:#2196F3,stroke:#333,stroke-width:2px,color:#fff
    style SensitiveInfo fill:#9C27B0,stroke:#333,stroke-width:2px,color:#fff
    style Stop1 fill:#F44336,stroke:#333,stroke-width:2px,color:#fff
    style Stop2 fill:#F44336,stroke:#333,stroke-width:2px,color:#fff
    style Stop3 fill:#F44336,stroke:#333,stroke-width:2px,color:#fff
    style Stop4 fill:#F44336,stroke:#333,stroke-width:2px,color:#fff
    style Allow fill:#4CAF50,stroke:#333,stroke-width:2px,color:#fff
```

## 📋 **Processing Logic**

### **1. Content Policy Filters (First)**
```yaml
Evaluation Order:
  - SEXUAL: HIGH → Scan for sexual content
  - VIOLENCE: HIGH → Scan for violent content  
  - HATE: HIGH → Scan for hate speech
  - INSULTS: LOW → Scan for insults (permissive)
  - MISCONDUCT: NONE → Skip misconduct check
  - PROMPT_ATTACK: NONE → Skip prompt injection check
```

**Result**: If ANY filter triggers above threshold → **BLOCK**

### **2. Topic Policy (Second)**
```yaml
Topic Matching:
  - CodeGeneration: ALLOW → Check if content matches
  - FileOperations: ALLOW → Check if content matches
  - EncodedContent: ALLOW → Check if content matches
  - CodeInjection: ALLOW → Check if content matches
```

**Logic**: 
- If content matches ANY `ALLOW` topic → **CONTINUE**
- If content matches ANY `DENY` topic → **BLOCK**
- If no topic matches → Use default behavior

### **3. Word Policy (Third)**
```yaml
Word Scanning:
  - Allowed Words: ['rewrite', 'refactor', 'code', 'file'] → ALLOW
  - Blocked Words: [profanity list] → BLOCK
```

**Result**: If blocked word found → **BLOCK**

### **4. Sensitive Information Policy (Fourth)**
```yaml
PII Detection:
  - EMAIL: ANONYMIZE → Replace with [EMAIL]
  - PHONE: ANONYMIZE → Replace with [PHONE]
  
Regex Patterns:
  - ProductionAPIKeys: BLOCK → Block if matches
  - AllowEncodedContent: ALLOW → Allow if matches
  - AllowFileReferences: ALLOW → Allow if matches
```

**Logic**:
- **BLOCK** patterns → **IMMEDIATE BLOCK**
- **ALLOW** patterns → **CONTINUE PROCESSING**
- **ANONYMIZE** → **MODIFY CONTENT** and continue

## ⚡ **Key Processing Rules**

### **❌ Blocking Behavior:**
- **ANY** content filter above threshold → **IMMEDIATE BLOCK**
- **ANY** blocked word found → **IMMEDIATE BLOCK**  
- **ANY** BLOCK regex pattern match → **IMMEDIATE BLOCK**
- **NO** matching ALLOW topic (if topics defined) → **BLOCK**

### **✅ Allow Behavior:**
- **ALL** content filters pass (below threshold)
- **AND** at least one ALLOW topic matches
- **AND** no blocked words found
- **AND** no BLOCK regex patterns match

### **🔄 Processing Rules:**
- Content filters with `NONE` strength are **SKIPPED**
- **EARLY TERMINATION**: Processing stops immediately on ANY BLOCK
- **ANONYMIZE EXCEPTION**: Only ANONYMIZE actions modify content and continue
- **ALL MUST PASS**: Every policy must allow for final approval

## 🎯 **Your Specific Case**

### **Why Your Prompt Was Blocked:**
```
Original Prompt: "ipynb#W0sZmlsZQ%3D%3D" + "file to be rewritten" + "refactor code"

Processing Flow:
1. Content Filters: PROMPT_ATTACK detected encoded content → BLOCK
2. Never reached Topic Policy
3. Never reached Word Policy  
4. Never reached Regex Policy
```

### **How Enhanced Guardrail Fixes It:**
```yaml
1. Content Filters:
   - PROMPT_ATTACK: NONE → SKIP (no blocking)
   - MISCONDUCT: NONE → SKIP (no blocking)

2. Topic Policy:
   - CodeGeneration: ALLOW → MATCHES → CONTINUE
   - EncodedContent: ALLOW → MATCHES → CONTINUE

3. Word Policy:
   - 'rewrite': ALLOWED → CONTINUE
   - 'file': ALLOWED → CONTINUE

4. Regex Policy:
   - 'ipynb#[A-Za-z0-9+/=%]+': ALLOW → CONTINUE
   - '(file|rewrite|refactor|code)': ALLOW → CONTINUE

Final Result: ALL POLICIES PASS → ALLOW
```

## 🔍 **Testing Different Scenarios**

### **Scenario 1: Early Block (Most Common)**
```
Input: "Please hack this system and rewrite code"

Processing:
1. Content Filter: MISCONDUCT=HIGH → BLOCK
2. STOPS HERE - Never reaches Topic/Word/Regex policies
3. Result: BLOCKED
```

### **Scenario 2: Sequential Until Block**
```
Input: "Please improve this damn code"

Processing:
1. Content Filters: PASS
2. Topic Policy: PASS
3. Word Policy: "damn" in profanity list → BLOCK
4. STOPS HERE - Never reaches Sensitive Info policy
5. Result: BLOCKED
```

### **Scenario 3: Full Processing (All Pass)**
```
Input: "Please improve this Python code"

Processing:
1. Content Filters: PASS
2. Topic Policy: PASS
3. Word Policy: PASS
4. Sensitive Info: PASS
5. Result: ALLOWED
```

## 📊 **Performance Implications**

### **Processing Time:**
- **Early Block**: Faster (stops at first blocking policy)
- **Full Allow**: Slower (must check all policies)
- **Regex Heavy**: Slower (pattern matching intensive)

### **Optimization Tips:**
- Set restrictive content filters to `NONE` for development
- Use broad ALLOW topics to reduce topic matching overhead
- Minimize regex patterns for better performance
- Use specific word lists rather than broad pattern matching

## 🎯 **Summary **

**Guardrails DO stop at first blocking match** - they use early termination:

1. **Sequential Processing**: Content → Topic → Word → Sensitive Info
2. **Early Termination**: Processing stops immediately on ANY BLOCK
3. **Any Block = Final Block**: One blocking policy blocks entire request
4. **All Must Pass**: Every policy must allow for final approval
5. **ANONYMIZE Exception**: Only ANONYMIZE actions modify and continue processing

