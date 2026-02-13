# Vernacular (Corrected)

## Terms

| Term | Meaning | NOT |
|------|---------|-----|
| **Researching** | Context gathering | "digging" |
| **Questioning** | Interview-driven claim extraction | - |
| **Crystallizing** | Structure from chaos | - |
| **Committing** | Approve spec | - |
| **Planning** | Sequence tasks | - |
| **Executing** | Run BD tasks | "manifesting" (woo) |
| **View** | Rendered projection (markdown) | NOT a type |
| **Claim** | Atomic assertion | - |
| **Evidence** | Proof for claim | - |
| **Gate** | Validation checkpoint | - |
| **Ephemeral** | STATE not entity type | - |

## Lifecycle States (Clean)

```
ephemeral → researching → crystallizing → committed → planning → executing → complete
```

## Command Mapping

| State | Command |
|-------|---------|
| researching | `/research` |
| questioning | `/interview` |
| crystallizing | `/work design` |
| committed | `/work spec` |
| planning | `/work plan` |
| executing | `lev exec` / `bd` |
