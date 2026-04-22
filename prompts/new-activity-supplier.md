# Prompt — Onboard a New Activity Supplier

Use this prompt to generate all ATP artefacts required to onboard a new activity supplier into the kernel.

---

## How to use

Copy the prompt below into your Claude Code session, filling in the supplier details in the `[SUPPLIER]` block.

---

## Prompt

```
I need to onboard a new activity supplier into the ATP Kernel.

Read CLAUDE.md and ARCHITECTURE.md before proceeding.

The supplier details are:

[SUPPLIER]
- Name: 
- Legal name (Japanese if applicable): 
- Party ID (lowercase-hyphenated, e.g. "hakuba-ski-school"): 
- Regulatory class: FARM_EXPERIENCE | OWN_SUPPLY (choose one)
- Activity category: (e.g. SKI_ALPINE, FARM_EXPERIENCE, HIKING — see @atp/types ActivityCategory)
- Location: city, prefecture, Japan
- Activities offered: (list each with name, duration in minutes, max participants, price in JPY)
- Operating days/season: 
- Notification channel (supplier-facing): LINE
- Languages: 
- Trust basis with MyAuberge: INTER_ENTITY_SERVICE_AGREEMENT | THIRD_PARTY_KYC (choose one)

[/SUPPLIER]

Generate the following files:
1. fixtures/party-registry/[party-id].json — Party Registry entry
2. fixtures/capability-declarations/[party-id]-capability.json — Capability Declaration
3. fixtures/trust-chains/myauberge-[party-id]-trust.json — Trust Chain Declaration (with MyAuberge as trusting party)

Follow the structure of the existing fixtures in the fixtures/ directory exactly.

Mark any missing information as PLACEHOLDER with the relevant OQ number if known, or a descriptive note.

After generating the fixtures, show me what Party Registry registration code I need to add to packages/core/src/party-registry/ to load this supplier on kernel startup.
```

---

## Notes

- Activity categories must come from the ATP Activity Category Registry — see `@atp/types` `ActivityCategory` type
- Org-namespaced extensions use the format `atp_{org}_{activity}` (e.g. `atp_myauberge_sake_tasting`)
- The `regulatory_class` drives Cedar policy set selection — use `FARM_EXPERIENCE` for independent activity suppliers and `OWN_SUPPLY` only for operators who both own the accommodation and the activity
- Trust basis `INTER_ENTITY_SERVICE_AGREEMENT` requires both parties to have the same principal — suitable for Ponyhouse Farm. Third-party suppliers use `THIRD_PARTY_KYC`
