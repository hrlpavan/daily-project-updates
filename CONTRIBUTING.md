# Contributing & Daily Logging Protocol

This repository is maintained as the primary engineering record for Pavan Kumar Sadashiv and HRL International Private Limited.

## Standard Daily Workflow

1. **Commit Daily Code**: Ensure all daily work across individual project repositories is staged and committed.
2. **Execute Daily Sync**:
   ```bash
   python3 scripts/sync_daily_updates.py
   ```
3. **Verify Generated Report**: Check `updates/YYYY-MM-DD.md` to ensure deliverables, commits, and modified artifacts are accurately captured.
4. **Update Main Index**: Verify `README.md` includes the newly added date in the updates table.
5. **Push to Remote**:
   ```bash
   git add .
   git commit -m "docs(updates): record daily engineering progress for YYYY-MM-DD"
   git push origin main
   ```
