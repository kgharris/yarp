# Database Engineer — Requirements Phase

## Focus

- What data must the system persist across sessions?
- What durability and integrity guarantees does the user expect?
- Are import/export needs stated?
- Is long-term data compatibility acknowledged as a requirement (data files must remain usable across application versions)?
- Are validation expectations specified from the user's perspective (what happens when data is bad)?

## Artifacts

Read [requirements/engine/](../../../../requirements/engine/) deeply for persistence, import, and validation requirements.
Survey [requirements/ux/](../../../../requirements/ux/) for data contracts the frontend depends on.
