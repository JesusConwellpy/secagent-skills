# Persona Cards Reference

## Persona Card Structure

```rust
pub struct PersonaCard {
    pub name: String,
    pub persona_type: PersonaType,    // Security / General
    pub expertise: Vec<String>,       // Domain expertise keywords
    pub tone: String,                 // Voice/tone description
    pub constraints: Vec<String>,     // Scope boundaries
    pub forbidden_actions: Vec<String>, // NEVER do these
    pub tool_preferences: Vec<String>,  // Preferred tools
    pub max_autonomy: AutonomyLevel,    // Low / Medium / High
    pub team_role: TeamRole,           // Recon / WebAnalyst / BinaryAnalyst / ...
    pub system_prompt: String,         // Full sub-agent system prompt
    pub fingerprint: String,           // SHA256 integrity hash
}
```

## Persona Injection Format

```xml
<persona_card name="recon" fingerprint="sha256:abc123...">
  ## Role: Reconnaissance Specialist
  **Expertise**: port scanning, service discovery, OSINT
  **Tone**: calm, precise

  ## Constraints
  - Reconnaissance only — no exploitation
  - Report all findings, suppress nothing
  - Verify every open port with service fingerprinting

  ## Forbidden Actions
  - NEVER execute exploits
  - NEVER modify target systems
  - NEVER exfiltrate data

  ## Tool Preferences
  - nmap (preferred)
  - curl, wget
  - whois, dig, nslookup
</persona_card>
```

## Team Designer Blueprint

```rust
pub struct TeamBlueprint {
    pub name: String,
    pub objective: String,
    pub members: Vec<TeamMember>,
}

pub struct TeamMember {
    pub persona: String,       // Persona name to use
    pub count: usize,          // How many instances
    pub assignment: String,    // What this member does
    pub depends_on: Vec<String>, // Which members must complete first
}
```

## TeamRole Enum

```rust
pub enum TeamRole {
    Recon,           // Reconnaissance specialist
    WebAnalyst,      // Web application analyst
    BinaryAnalyst,   // Binary/reverse engineering
    ExploitRunner,   // Exploit execution
    IntelGatherer,   // Intelligence/cve research
    CodeAuditor,     // Source code security review
    PostExploit,     // Post-exploitation
    Coordinator,     // Team coordination
    Custom,          // Custom role
}
```
