# Acceptance Criteria

A submission is marked complete when all items below are met.

## Submission structure
- Work is submitted in `cloud-devops-submissions` under `week-02/`.
- `notes.md`, `commands.txt`, and a `screenshots/` folder are included.

## Part 1 – Linux (processes + SSH service + logs)
- `commands.txt` shows the commands were run:
  - `ps aux | head`
  - `ps aux | grep ssh`
  - `top`
  - `systemctl status ssh`
  - `sudo journalctl -u ssh --since "30 minutes ago"`
- `notes.md` includes:
  - one observation from `systemctl status ssh`
  - one observation from the `journalctl` output
- Screenshot evidence includes `ps aux | grep ssh`.

## Part 2 – Script + PM2
- `commands.txt` shows:
  - `chmod +x` was used on the start script
  - PM2 was installed
  - app was started with PM2 using the agreed command
  - `pm2 list` was checked
- Screenshot evidence includes `pm2 list`.

## Part 3 – Git (fork + branch + PR)
- Student created a branch `feature/add-name-to-readme`.
- `README.md` was updated with the student name under a Contributors/Training section.
- A PR was opened and merged in the student fork.
- Screenshot evidence includes the merged PR page.
