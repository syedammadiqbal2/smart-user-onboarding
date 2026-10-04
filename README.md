# Smart User Onboarding

Smart User Onboarding - Free Salesforce AppExchange Application.

Create internal Salesforce users consistently by using a trusted existing user, or an activity-ranked user from a Role, as an editable access template. The app suggests Profile, Role, Permission Set Licenses, Permission Sets, Permission Set Groups, Queues and Public Groups, validates every selection on the server, provisions the user, and records an audit trail.

> **Status:** in development. Not yet packaged or published.

## Development

Requires Node.js 18+, the Salesforce CLI (`sf`) and Git.

```bash
npm install
sf project deploy start --target-org <your-dev-org>
sf apex run test --code-coverage --result-format human --target-org <your-dev-org>
```

Source lives in `force-app/`. The project is developed without a namespace; the managed-package namespace is added at packaging time.

## License

MIT
