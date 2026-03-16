# Contributing

44Mesh is an open project. Contributions are welcome in the form of bug reports, documentation improvements, code contributions, and new features.

---

## Repository Structure

```
44mesh/
├── deploy/
│   ├── bird-border/    # Border router (BIRD2 + ZeroTier controller)
│   ├── zerotier/       # Mesh node client
│   ├── zerotier-ui/    # Web management interface
│   └── rpi-isp/        # Mock ISP for testing
├── docs/
│   └── ZEROTIER.md     # ZeroTier architecture guide
└── .github/
    └── workflows/      # CI/CD pipelines
```

---

## How to Contribute

### Bug Reports

Open an issue at the GitHub repository with:
- What you expected to happen
- What actually happened
- Steps to reproduce
- Relevant logs (`docker compose logs`, `birdc show protocols`, etc.)
- Your configuration (with secrets redacted)

### Documentation

Documentation improvements are always welcome. Fix typos, clarify confusing sections, add examples, or document undocumented behavior.

### Code Contributions

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-improvement`
3. Make your changes
4. Test with the mock ISP if the change affects routing
5. Validate Docker Compose configs: `docker compose config`
6. Open a pull request

---

## CI/CD Pipeline

Pull requests trigger automatic validation:

- `docker compose config` validation for all components
- Docker build without cache for the modified component
- Results reported in the PR check summary

All checks must pass before merging.

---

## Testing Changes Locally

Use the mock ISP for end-to-end testing without a real BGP peer:

```bash
# Start border router
cd deploy/bird-border && docker compose up -d

# Start mock ISP
cd deploy/rpi-isp && docker compose up -d

# Verify BGP session
docker exec bird-border birdc show protocols

# Start a mesh node
cd deploy/zerotier && docker compose up -d

# Authorize the node and test connectivity
```

---

## AlterMundi ZeroTier Fork

The custom ZeroTier client is maintained separately at `github.com/AlterMundi/ZeroTierOne`, branch `feature/ingress-node`. Contributions to the fork's ingress routing feature should be submitted there.

---

## Code of Conduct

This project follows the contributor covenant code of conduct. Be respectful, constructive, and collaborative.
