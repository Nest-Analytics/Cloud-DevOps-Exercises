# Acceptance Criteria

Submission is marked complete when all items below are met.

## Submission structure
- Work is submitted in `cloud-devops-submissions` under `week-03/`.
- `notes.md` is included.
- `screenshots/` folder is included.

## Notes content
`notes.md` includes:
1. Private IP captured from `hostname -I`.
2. Public IP captured from the portal.
3. Results summary covering both scenarios:

**When port 3000 was allowed in the NSG**
- `curl http://localhost:3000` from VM result is stated.
- Browser access to `http://PUBLIC_IP:3000` from laptop result is stated.

**When port 3000 was blocked in the NSG**
- `curl http://localhost:3000` from VM result is stated.
- Browser access to `http://PUBLIC_IP:3000` from laptop result is stated.

## Screenshot evidence
Inside `week-03/screenshots/`:
- A screenshot showing `pm2 list` output with the app visible.
- A screenshot showing the NSG inbound rule that allows TCP 3000 (rule name and port visible).

## Outcome demonstrated
The submission clearly demonstrates:
- local access (`localhost:3000`) works independently of NSG rules
- external access (`PUBLIC_IP:3000`) depends on the NSG inbound rule for port 3000
