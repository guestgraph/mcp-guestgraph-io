# mcp.guestgraph.io

GuestGraph's own model, `guestgraph/mental-model`, served over MCP at `https://mcp.guestgraph.io/mcp`. That model is the project behind the open-source guest identity graph, described in CompanyGraph's vocabulary, with no person and no guest's data anywhere in it. This repository pins one commit of the model and one release of `companygraph/mcp-server`, builds an image that carries the model's snapshot, and runs it on Cloud Run in Zurich behind Firebase Hosting. Everything below the Google Cloud project is Terraform, applied by GitHub Actions.

It is the third deployment of the server, after `robertblust/mcp-blust-ch` and `companygraph/mcp-companygraph-io`, and it is built entirely from the parts the server ships under `deploy/`: the Terraform modules, the build command, the shared tests and the workflows. What is this deployment's own is its values in `deployment.json`, its brand, its page styles and its instance tests.

## Using it

Add `https://mcp.guestgraph.io/mcp` as a custom connector in Claude, or as a remote MCP server in ChatGPT's developer mode or the Gemini CLI. No authentication. The page at `https://mcp.guestgraph.io` lists the tools with what each returns, from the server's own list: the types and their schemas, what the types declare about each other, the rules, and the entities with their references. Every answer names the model commit it was read from.

## What pins what

`source.json` names the model commit and `package.json` the server release. Moving either is a pull request; the merge builds the image, applies the infrastructure with it and checks that the service reports the new commit.

## Building it

    npm ci
    npm run snapshot      # writes dist/snapshot.json from the pinned commit
    npm run page-css      # writes dist/page.css from the design package's blocks and own.css
    npm run jsonld        # writes dist/jsonld.json from the snapshot
    npm test              # the server's shared deployment tests and this instance's own
    docker build -t mcp-guestgraph-io:local .

## Infrastructure

`infra/bootstrap/` is applied by the owner, with the state `infra/bootstrap/README.md` says how to restore, and holds what CI needs before it can authenticate: the state bucket, the identity pool, the service accounts and the image registry. A pull request plans as `terraform-plan@`, which reads and changes nothing, and only a run on `main` may apply as `terraform@` or push as `deploy@`. `infra/` is applied by CI on every merge: its state in the bucket the bootstrap made, and one call into the module `companygraph/mcp-server` ships under `deploy/terraform`, with this deployment's own values read from `deployment.json`. `deploy.yml` only calls the package's own `deployment.yml`, by the release `package.json` pins. `publish.yml` runs the steps of the package's `registry.yml` itself, because a workflow in another organization receives no secret, and the Registry's signing key has to reach the job; its comment says to keep the steps in step with the release.

Publishing to the MCP Registry runs in the `registry` environment, which requires the owner's review of every run. The signing key lives there as an environment secret, `MCP_PRIVATE_KEY`, never as a repository secret, because a repository secret would be readable by any workflow on any branch and the review gate would protect nothing.

## The owner's steps

What the owner does once, because Terraform cannot do it or CI cannot yet sign in. They follow mcp.companygraph.io's, which stood up the same modules. The key commands need OpenSSL 3, and `/usr/bin/openssl` on macOS is LibreSSL, so every step runs in a shell that has first run:

    export PATH=/opt/homebrew/bin:$PATH

1. Create the project and link it to the billing account, then write the project number into `deployment.json` as `project_number`:

        gcloud projects create guestgraph-io-mcp --organization=14986580178
        gcloud billing projects link guestgraph-io-mcp --billing-account=011DEB-4A45A0-3A52BB
        gcloud projects describe guestgraph-io-mcp --format='value(projectNumber)'

2. Enable the Cloud Billing API on the new project:

        gcloud services enable cloudbilling.googleapis.com --project guestgraph-io-mcp

3. Apply the bootstrap before the merge, because the merge's deploy signs in with what it creates. Its state is a local file git ignores, and `git worktree remove` deletes ignored files without a word, so copy it out the moment the apply finishes, to `~/guestgraph-io-mcp-bootstrap.tfstate` and a second place that is safe, because it is the bootstrap's only state:

        gcloud auth application-default login
        terraform -chdir=infra/bootstrap init && terraform -chdir=infra/bootstrap apply
        cp infra/bootstrap/terraform.tfstate ~/guestgraph-io-mcp-bootstrap.tfstate

4. Open the chat's state, on the pull request's branch, under the owner's login, because a pull request's read-only plan identity may read it but never create it:

        terraform -chdir=infra/chat init

5. Give the chat its key, from a workspace of its own in the Anthropic Console with a monthly spend limit. The secret is made once, the key goes in on stdin and never in a file, and the read access is granted once the merge's apply has made the runtime account `chat-run@`:

        gcloud services enable secretmanager.googleapis.com --project guestgraph-io-mcp
        gcloud secrets create chat-anthropic-key --replication-policy automatic --project guestgraph-io-mcp
        printf '%s' "$KEY" | gcloud secrets versions add chat-anthropic-key --data-file=- --project guestgraph-io-mcp
        gcloud secrets add-iam-policy-binding chat-anthropic-key --member serviceAccount:chat-run@guestgraph-io-mcp.iam.gserviceaccount.com --role roles/secretmanager.secretAccessor --project guestgraph-io-mcp

6. Merge the pull request. The first deploy of each service fails at its live check by design, and the apply's warning names the host Cloud Run gave it; write them into `deployment.json` and `chat/chat.json` as `run_host` and merge that.

7. At Hostpoint, add the records each service names, replacing the default records of `mcp.guestgraph.io` and `chat.guestgraph.io`:

        terraform -chdir=infra init && terraform -chdir=infra output dns_records
        terraform -chdir=infra/chat init && terraform -chdir=infra/chat output dns_records

8. Send one message by hand, because the deploy's `GET /chat` proves the route and the host and never the model:

        curl -N -H 'X-Chat: 1' -H 'content-type: application/json' https://chat.guestgraph.io/chat -d '{"messages":[{"role":"user","content":"What does GuestGraph never do with a source record?"}],"lang":"en"}'

9. For the Registry, make the signing key outside the repository and store the private key as `MCP_PRIVATE_KEY` in the `registry` environment, which the fourth and fifth commands create with the owner as its required reviewer and tags `v*` as the only refs that may deploy to it. The third command prints the TXT record to publish at the apex of `guestgraph.io`. Move `"$K/key.pem"` into a password manager before the last command if the key is to be kept:

        K=$(mktemp -d)
        openssl genpkey -algorithm Ed25519 -out "$K/key.pem"
        echo "guestgraph.io. IN TXT \"v=MCPv1; k=ed25519; p=$(openssl pkey -in "$K/key.pem" -pubout -outform DER | tail -c 32 | base64)\""
        echo '{"reviewers":[{"type":"User","id":7037057}],"deployment_branch_policy":{"protected_branches":false,"custom_branch_policies":true}}' | gh api -X PUT repos/guestgraph/mcp-guestgraph-io/environments/registry --input -
        gh api -X POST repos/guestgraph/mcp-guestgraph-io/environments/registry/deployment-branch-policies -f name='v*' -f type=tag
        openssl pkey -in "$K/key.pem" -noout -text | grep -A3 'priv:' | tail -n +2 | tr -d ' :\n' | gh secret set MCP_PRIVATE_KEY --env registry --repo guestgraph/mcp-guestgraph-io
        rm -rf "$K"

10. Once the server is live and the model's surfaces are merged, tag the deployment's version, `v0.1.0` for the first entry, on `main` and approve the `registry` run:

        git fetch origin && git tag v0.1.0 origin/main && git push origin v0.1.0

11. Grant the owner's own login the reading of the chat's questions and reports, once, since the address belongs in no repository; a read then impersonates the analyst, with the commands the chat server's README gives:

        gcloud iam service-accounts add-iam-policy-binding chat-analyst@guestgraph-io-mcp.iam.gserviceaccount.com --member user:<login> --role roles/iam.serviceAccountTokenCreator --project guestgraph-io-mcp

## The chat

`chat.guestgraph.io` is the chat over this host: a visitor's question on guestgraph.io goes to it, it asks `mcp.guestgraph.io` through the tools, and Claude Sonnet 5 on the Anthropic API writes the answer from what the tools said. It is `companygraph/chat-server`, deployed from this repository beside the server: `chat/` holds what is this deployment's own for it, `chat/package.json` pinning the release, `chat/chat.json` naming the domain, the host it reads, the page origins it answers, the provider and the month's ceiling in input-equivalent tokens, the Dockerfile, the brand, the stylesheet and the tests; `infra/chat/` is its Terraform root, applied by CI with its own state prefix in the same bucket; `.github/workflows/chat.yml` calls the chat server's own workflow, and `.github/workflows/report.yml` calls its report workflow, on Monday at six UTC and by hand. The release is named in those four places and the chat server's pin test holds them to one. The budget in `deployment.json` covers both services.

Nothing the chat spends escapes its ceiling. The service refuses before it asks the model, an address gets twenty requests an hour, and a meter in this project's Firestore database counts every model call against a day's share and a month's ceiling, the same ceiling as chat.companygraph.io's. The chat is stopped by hand where the meter keeps it: the `(default)` database, document `chat/meter`, field `closed` set to `true`, which refuses the next message and spends nothing, and back to `false` to open it; the day and the month there are UTC. The Anthropic Console's own monthly spend limit on the key is a second stop outside this project.

Once a message the shape accepted has its answer or its refusal, the chat keeps one line of it, the question, the language and what the loop saw, no address and no word of the answer, in this project's log bucket `chat-questions` for ninety days, behind the view `questions` that one account, `chat-analyst`, may read and nothing else; the project's console finds those lines as `labels.logger="chat.question"`. On Monday morning the report workflow, running as that account, writes the week that ended to the private bucket `chat-reports-guestgraph-io-mcp` as `reports/<week>.md`: how many questions, how many answered, by language, the entities most cited and the unanswered questions in full. A report is deleted at eighty-three days, so no question it quotes outlives the ninety, which is the one number guestgraph.io's privacy page gives.

## License

CC BY 4.0 for the text here; the model's own license is its own.
