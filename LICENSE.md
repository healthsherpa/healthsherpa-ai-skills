# License

MIT License

Copyright (c) 2026 HealthSherpa

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

---

## AI Usage Disclaimer

The skills in this repository are designed to provide context to AI coding
assistants (such as Cursor, GitHub Copilot, and similar tools). By using
these skills, you acknowledge and agree to the following:

**No guarantee of correctness.** AI-generated code based on these skills may
contain errors, omissions, or misinterpretations. All output must be reviewed,
tested, and validated by a qualified developer before use in any environment.

**Not a substitute for official documentation.** These skills supplement the
official HealthSherpa API documentation at
[docs.ichra.healthsherpa.com](https://docs.ichra.healthsherpa.com). In case
of any discrepancy, the official documentation is authoritative.

**No liability for AI output.** HealthSherpa is not responsible for any code,
decisions, or actions taken based on AI-generated output produced using these
skills. Use at your own risk.

**Health data and compliance.** Integrations built with the assistance of
these skills may handle protected health information (PHI) or personally
identifiable information (PII). You are solely responsible for ensuring your
implementation complies with all applicable laws and regulations, including
HIPAA, state privacy laws, and CMS requirements.

**Integrator security responsibility.** You are solely responsible for the
security of your systems, networks, application code, API credentials
(including the HealthSherpa API key, webhook signing secret, and staging
Basic Auth credentials), webhook endpoints, and any data you collect, store,
transmit, or process — including all PII and PHI handled in the course of
using HealthSherpa APIs. HealthSherpa is not responsible for credentials
leaked through your source control, client-side code, logs, error reports,
screenshots, or any other artifact produced by you, your developers, or AI
coding assistants operating on your behalf. Review and audit all generated
code for credential exposure, missing input validation, missing output
encoding, missing webhook signature verification, and unsafe logging before
deploying to any environment.

**Skills may not reflect the latest API state.** API behavior, endpoints,
field names, and carrier-specific logic change over time. Always verify
against the live API and current documentation before deploying.
