# Runbook: Temporal Wordflow

This runbook is for presenting the demo. The [README](README.md) explains what it is and how it's built.

## What it shows

Wordflow is a word-crossword game whose whole backend state lives in Temporal Workflows. Each player, each level a player plays, each campaign, the campaign catalog, and the leaderboard is a long-lived Workflow. There is no application database, cache, or object store, and the web API holds nothing between requests.

The demo makes three problems visible:

- **A stateful app needs somewhere durable to keep its state.** Accounts, progress, points, and in-progress games normally sit in a database with a service in front of it. Here each one is a Workflow you can open and read. Beats 1, 2, and 3 show it.
- **Every request must apply exactly once, even when something fails or retries.** A guess, a hint, or a point purchase can't be lost or counted twice, including when one entity changes another. Beats 2, 3, and 4 show it.
- **Entities that live for months still have to take code changes.** The Workflows are pinned to a Worker Deployment Version and move to new code only at a Continue-As-New boundary. Beat 6 shows it on tmprl-demo.cloud.

## Before you start

- Install Go 1.26 or later and the [Temporal CLI](https://docs.temporal.io/cli).
- Clone the repo. The first `go run` downloads the dependencies.
- The Worker and the web server load their Temporal connection with the Go SDK's environment configuration, which also reads your Temporal CLI config file. If your default CLI profile points at Temporal Cloud, the local run picks up its API key and fails with a TLS handshake error. Point both terminals at an empty config file before you start them:

  ```bash
  touch /tmp/wordflow-empty.toml
  export TEMPORAL_CONFIG_FILE=/tmp/wordflow-empty.toml
  ```

- No model or other paid API is needed. The daily challenge generator calls OpenAI, but nothing starts it unless you run `cmd/daily-challenge`, and this runbook doesn't use it.

### On tmprl-demo.cloud

The catalog entry ([`temporal-wordflow`](https://github.com/temporal-sa/tmprl-demo-cloud-registry/blob/main/projects/demo/temporal-wordflow.yaml)) runs the same code against a project Temporal Cloud Namespace:

- `web` is the game and API (`cmd/server`), at https://games.tmprl-demo.cloud. It needs no Temporal login to play.
- `worker` is a Serverless Worker on AWS Lambda (`cmd/lambda-worker`), and `steady` is a long-running Worker (`cmd/worker`). Both poll the `temporal-word-game` Task Queue as Worker Deployment `temporal-word-game`. The `steady` Worker takes its Build ID from the platform's rollout version.
- The platform rolls out each change to the repo's `main` branch as a new Worker Deployment Version.
- The workers aren't crashable from the platform, so beat 3's "stop the Worker" step is local only.

## Start it

In three terminals, from the repository root:

1. `make temporal` (skip this on tmprl-demo.cloud). It keeps the dev server's database in `.temporal/`.
2. `make worker`
3. `make server`

Open http://localhost:8080. A healthy start shows the **Log in** and **Sign up** cards under the **Temporal Wordflow** header. The Worker terminal logs `Started Worker ... TaskQueue temporal-word-game` and then a `RegisterCampaign` Activity, and the server terminal logs `web app listening on :8080`. The server starts the `Temporal Foundations` campaign at startup, so the Temporal UI at http://localhost:8233 already lists `wordflow-campaign/temporal-foundations` and `catalog/global` as Running.

Keep three more tabs ready:

- http://localhost:8080/leaderboard, the public leaderboard.
- http://localhost:8080/workflows/, the read-only Workflow Explorer. The **Player Workflow ↗**, **Game Workflow ↗**, and **Campaign Workflow ↗** links in the game open it.
- The Temporal UI, for anyone who wants the raw Event History.

### Answers for the first levels

So you don't have to solve puzzles live:

| Level | Letters | Answers |
|---|---|---|
| 1 · Workflow Flow | FLOW | FLOW, WOLF, LOW, OWL |
| 2 · Workflow Time | EIMT | TIME, ITEM, EMIT, TIE |
| 3 · Durable State | AESTT | STATE, TASTE, SEAT, EAST, TEA, SET |

The rest are in [`internal/bootstrap/temporal-foundations.json`](internal/bootstrap/temporal-foundations.json).

## The walkthrough

To spell a word, click the letters in the circle, then **Submit word**. **Clear** empties the word and **Shuffle** reorders the letters. The game has no keyboard shortcuts.

### 1. Sign up, and a Workflow becomes the account

1. In **Sign up**, enter a username, a display name, a password, and the same password in **Confirm password**. Press **Create account**. Use a new username for every presentation.
2. The page switches to the player view: five stats (**current streak**, **best streak**, **points**, **lifetime earned**, **levels completed**), the **Streak freeze** shop, and the **Temporal Foundations** campaign card with levels 1 to 3 open and the rest locked.
3. Click **Player Workflow ↗** in the header. The Workflow Explorer opens on `player/<username>`, a `PlayerWorkflow` that started a moment ago. Its timeline shows the start and one accepted Update, `open-session`.
4. Say: "There's no users table. Signing up started a Workflow whose ID is my username, and that Workflow holds my password hash, points, and progress. The API in front of it keeps nothing."
5. If someone asks what stops two people taking the same name, try **Sign up** again with the same username and a different password. The page shows `username is already registered with a different password`.

### 2. Play a level, and each guess is an Update

1. Back in the game tab, press **Start campaign**. Level 1, **Workflow Flow**, opens with the letters F, L, O, and W.
2. Spell `WOLF` and press **Submit word**. The notice reads **Word recorded in history.** and the word fills in on the grid. Spell `FOL`; the notice reads **Not in the wordflow.** and the word appears in the **Not in the wordflow** list beside the board.
3. Point at the **Bonus points** panel. The speed, hint, and accuracy bonuses are computed in the level Workflow, and the accuracy bonus just dropped by 2.
4. Click **Game Workflow ↗**. The Explorer opens `wordflow-level/<username>/temporal-foundations/1`, a child of the Player Workflow. Each guess is a `submit-guess` Update in its timeline.
5. Say: "The level is its own Workflow. Each guess is a request to that Workflow that returns the new board, so the browser never computes game state."
6. Finish the level with `FLOW`, `LOW`, and `OWL`. The card reads **Workflow completed · Execution succeeded.** with the points breakdown.

### 3. Stop the Worker mid-level, and nothing is lost (local only)

1. Press **Play level 2**. Spell one word, such as `TIME`, and submit it.
2. Stop the Worker with `Ctrl-C` in its terminal.
3. Spell `ITEM` and press **Submit word**. The button reads **Submitting…** and stays that way, because no Worker is running the Workflow.
4. Start the Worker again with `make worker`. Within a few seconds of the Worker starting, the guess lands and the notice reads **Word recorded in history.**
5. Reload the page. You're still logged in, and level 2 comes back with both words in place.
6. Say: "The Worker was gone, and the request waited for it. When it came back, it picked up the level from its history and applied the guess once. Nothing in this app saved the board anywhere."

On tmprl-demo.cloud, skip steps 2 to 4. Reload the page or log out and back in instead: the level resumes where you left it, because it was never in the browser.

### 4. Buy a hint with points, across two Workflows

1. On level 2, the button reads **Reveal one · 2 free**. Press it twice; each press reveals a letter and counts down a free hint.
2. The button now reads **Reveal one · 10 points**. Press it. The notice reads **Spent 10 points · letter revealed**, and the player's points drop by 10.
3. Click **Game Workflow ↗**. The level's timeline shows a `SpendPoints` Activity. Click **Player Workflow ↗**; the player's timeline shows the matching `spend-points` Update.
4. Say: "The level owns the hint and the player owns the points. The level asks the player to deduct 10 points, and the request carries an ID made from the level's own request. If that call retries, Temporal hands back the first answer instead of charging me twice."

You need points for this beat, so play it after finishing level 1. A new player has 0 points, and a paid hint before the first win fails with `not enough points; need 10, have 0`.

### 5. The leaderboard updates on its own

1. Finish level 2 with the remaining answers.
2. Switch to the leaderboard tab. Your display name, with its `#` tag, appears with your lifetime points and levels completed. The page refreshes every 3 seconds.
3. Say: "When a level completes, the player publishes its new total to one leaderboard Workflow. That Workflow keeps the top 100 and ignores any total older than the one it has, so a retried publish can't add points twice."

### 6. Ship new code without breaking running players (tmprl-demo.cloud)

This beat needs a deployment on tmprl-demo.cloud, where every Workflow runs on a versioned Worker. The local `make worker` is unversioned.

1. In the Workflow Explorer, open your `player/<username>` Workflow. The strip under the metrics reads **Versioning** `PINNED`, **Deployment** `temporal-word-game`, and **Build** with the rollout's Build ID.
2. Say: "Every Workflow run is pinned to the build it started on, so a deploy never swaps code under a run that's in progress."
3. After a new rollout of `main` becomes current, wake the player with an Update: with no level open, press **Log out** and log in again. Open the Player Workflow again. It has continued as new, and the new run shows the new Build ID. An open level moves the same way at its next guess.
4. Say: "These players never finish, so they can't wait out a deploy. Each one already restarts its history with Continue-As-New, and that's the moment it moves to the new build. Nobody wrote a version check in the game code."

## Show it in code

All links point at commit [`beb6d77`](https://github.com/temporal-sa/temporal-demo-wordflow/commit/beb6d7799792b2dc8dadf57750a6d9c7372ed9a3). Open a file locally with `code -g <path>:<line>`.

### Make each player a Workflow, started by signing up

**When to use it:** after beat 1, or when someone asks where the users table is.

**Where it is:** [`internal/api/server.go` lines 276–302](https://github.com/temporal-sa/temporal-demo-wordflow/blob/beb6d7799792b2dc8dadf57750a6d9c7372ed9a3/internal/api/server.go#L276-L302) (`code -g internal/api/server.go:276`), handled in [`internal/workflows/player.go` lines 96–122](https://github.com/temporal-sa/temporal-demo-wordflow/blob/beb6d7799792b2dc8dadf57750a6d9c7372ed9a3/internal/workflows/player.go#L96-L122) (`code -g internal/workflows/player.go:96`)

```go
workflowID := workflows.PlayerWorkflowID(username)
var view game.PlayerView

if !register {
	err := s.update(ctx, workflowID, requestID, workflows.UpdateAuthenticatePlayer, update, &view)
	return view, rawToken, expiresAt, err
}

start := s.temporal.NewWithStartWorkflowOperation(client.StartWorkflowOptions{
	ID: workflowID, TaskQueue: s.taskQueue,
	WorkflowIDConflictPolicy: enums.WORKFLOW_ID_CONFLICT_POLICY_USE_EXISTING,
	WorkflowIDReusePolicy:    enums.WORKFLOW_ID_REUSE_POLICY_ALLOW_DUPLICATE_FAILED_ONLY,
}, workflows.PlayerWorkflowName, workflows.PlayerWorkflowInput{PlayerID: username})
handle, err := s.temporal.UpdateWithStartWorkflow(ctx, client.UpdateWithStartWorkflowOptions{
	StartWorkflowOperation: start,
	UpdateOptions: client.UpdateWorkflowOptions{
		UpdateID: requestID, UpdateName: workflows.UpdateAuthenticatePlayer,
		WaitForStage: client.WorkflowUpdateStageCompleted, Args: []any{update},
	},
})
```

```go
if err := workflow.SetUpdateHandlerWithOptions(ctx, UpdateAuthenticatePlayer,
	func(ctx workflow.Context, update AuthenticatePlayerInput) (game.PlayerView, error) {
		// ...
		if state.PasswordHash == "" {
			// The first accepted registration Update claims this username.
			state.PasswordHash = update.PasswordHash
			state.DisplayName = update.DisplayName
		}
		// ...
	}, workflow.UpdateHandlerOptions{
		Validator: func(_ workflow.Context, update AuthenticatePlayerInput) error {
			return validateCredentials(state, update)
		},
	}); err != nil {
```

**What you need to understand:**

- The username is the Workflow ID, `player/<username>`. Temporal allows one open Workflow per ID, which is what makes a username unique.
- Update-with-Start starts the Player Workflow if it isn't running and sends it the `open-session` Update in one call. The Go SDK requires a conflict policy for it; `USE_EXISTING` means a second signup for the same name reaches the existing Workflow instead of failing.
- The validator rejects a wrong password before the Update is accepted. A rejected Update writes nothing to the Workflow's history, so failed guesses at someone's password don't grow it.
- Log in uses a plain Update with no start, so an unknown username returns not-found and can't reserve a name.
- `ALLOW_DUPLICATE_FAILED_ONLY` lets the ID be reused only after a run that failed, timed out, was canceled, or was terminated. Terminating a Player Workflow frees its username.
- The password hash and session JWT are a deliberately small demo identity model. See [What's demo-only](#whats-demo-only).

**Docs:** [Update-with-Start](https://docs.temporal.io/develop/go/workflows/message-passing#update-with-start), [Entity Workflow pattern](https://docs.temporal.io/design-patterns/entity-workflow), [Workflow ID reuse policy](https://docs.temporal.io/workflow-execution/workflowid-runid#workflow-id-reuse-policy)

### Start a level as a child Workflow, inside the Update that asked for it

**When to use it:** during beat 2, when you open **Game Workflow ↗**, or when someone asks how one entity creates another.

**Where it is:** [`internal/workflows/player.go` lines 124–169](https://github.com/temporal-sa/temporal-demo-wordflow/blob/beb6d7799792b2dc8dadf57750a6d9c7372ed9a3/internal/workflows/player.go#L124-L169) (`code -g internal/workflows/player.go:124`)

```go
if err := workflow.SetUpdateHandlerWithOptions(ctx, UpdateStartLevel, func(ctx workflow.Context, update StartLevelInput) (StartLevelResult, error) {
	if err := lock.Lock(ctx); err != nil {
		return StartLevelResult{}, err
	}
	defer lock.Unlock()
	// ...
	activityCtx := workflow.WithActivityOptions(ctx, workflow.ActivityOptions{StartToCloseTimeout: 30 * time.Second})
	var resolution WordflowLevelResolution
	if err := workflow.ExecuteActivity(activityCtx, ActivityResolveWordflowLevel, ResolveWordflowLevelActivityInput{
		Query: WordflowLevelQuery{CampaignID: update.CampaignID, Player: playerProgress, Level: update.Level},
	}).Get(ctx, &resolution); err != nil {
		return StartLevelResult{}, fmt.Errorf("resolve campaign level: %w", err)
	}
	// ...
	levelID := WordflowLevelWorkflowID(state.PlayerID, update.CampaignID, update.Level)
	childCtx := workflow.WithChildOptions(ctx, workflow.ChildWorkflowOptions{
		WorkflowID: levelID,
	})
	future := workflow.ExecuteChildWorkflow(childCtx, WordflowLevelWorkflowName, WordflowLevelWorkflowInput{
		PlayerID: state.PlayerID, CampaignID: update.CampaignID, Puzzle: resolution.Puzzle, StartedAt: now,
	})
	// Wait only for Temporal to record the child start. Its completion stays
	// in the selector below.
	if err := future.GetChildWorkflowExecution().Get(ctx, nil); err != nil {
		return StartLevelResult{}, fmt.Errorf("start level workflow: %w", err)
	}
	levelFuture = future
	// ...
	return StartLevelResult{Player: playerView(state), Game: wordflowLevelView(initialGame)}, nil
```

**What you need to understand:**

- The `start-level` Update runs an Activity, starts a child Workflow, and returns the first board, all before the browser gets its answer. The UI shows the level in one round trip.
- The Activity asks the Catalog Workflow, with a Query, whether this level is unlocked for this player. Campaign rules live in one place, and the player doesn't call each campaign.
- The handler waits only for the child to start. The player's main loop picks up the child's result when the level ends, then adds points and updates the streak.
- The mutex keeps two Updates from interleaving while this one waits on the Activity. While a level is open, another `start-level` Update is rejected with `finish the active level before starting another`.
- The level's Workflow ID is `wordflow-level/<username>/<campaign>/<level>`, which is why the Explorer can show a player only their own levels.

**Docs:** [Child Workflows](https://docs.temporal.io/child-workflows), [Message handler patterns](https://docs.temporal.io/handling-messages#message-handler-patterns)

### Make every guess an Update, with the request ID as the Update ID

**When to use it:** after beat 2 or 3, or when someone asks what happens if the same request arrives twice.

**Where it is:** [`internal/api/server.go` lines 884–893](https://github.com/temporal-sa/temporal-demo-wordflow/blob/beb6d7799792b2dc8dadf57750a6d9c7372ed9a3/internal/api/server.go#L884-L893) (`code -g internal/api/server.go:884`). The browser makes the ID in [`internal/api/web/app.js` lines 710–717](https://github.com/temporal-sa/temporal-demo-wordflow/blob/beb6d7799792b2dc8dadf57750a6d9c7372ed9a3/internal/api/web/app.js#L710-L717).

```go
func (s *Server) update(ctx context.Context, workflowID, updateID, updateName string, input, output any) error {
	handle, err := s.temporal.UpdateWorkflow(ctx, client.UpdateWorkflowOptions{
		WorkflowID: workflowID, UpdateID: updateID, UpdateName: updateName,
		WaitForStage: client.WorkflowUpdateStageCompleted, Args: []any{input},
	})
	if err != nil {
		return err
	}
	return handle.Get(ctx, output)
}
```

```js
async function submitGuess() {
  const word = selectedIndexes.map((index) => letterOrder[index]).join("");
  if (!word) return;
  const result = await withGameMutation(elements.submit, "Submitting…", () =>
    api(`/api/me/campaigns/${currentGame.campaignId}/levels/${currentGame.level}/guesses`, {
      method: "POST",
      body: JSON.stringify({ requestId: requestID(), word }),
    }));
```

**What you need to understand:**

- Every change in the game goes through this one helper: signup, start level, guess, hint, streak freeze. Screens that only read use Queries.
- `WaitForStage: Completed` makes the HTTP call return the Update's result, which is the new board. That's why the button waits on **Submitting…** in beat 3: the Update waits for a Worker.
- The Temporal Service deduplicates Updates by Update ID within a Workflow run. If the same request ID arrives twice, the second call gets the first call's result and the handler doesn't run again.
- That deduplication is per run. The Player, Level, Catalog, and Leaderboard Workflows Continue-As-New, so a retry that lands on the new run would run again. The game has no ledger for that case.
- The browser makes a new ID for each click. Deduplication protects against a request delivered twice, such as a client or gateway retry, not against a player pressing the button twice.

**Docs:** [Message IDs and handling Continue-As-New](https://docs.temporal.io/handling-messages#exactly-once-message-processing), [Go: Send an Update](https://docs.temporal.io/develop/go/workflows/message-passing#send-update-from-client)

### Spend points across two Workflows exactly once

**When to use it:** after beat 4, or when someone asks how one entity changes another without a distributed transaction.

**Where it is:** [`internal/workflows/wordflow_level.go` lines 132–156](https://github.com/temporal-sa/temporal-demo-wordflow/blob/beb6d7799792b2dc8dadf57750a6d9c7372ed9a3/internal/workflows/wordflow_level.go#L132-L156) (`code -g internal/workflows/wordflow_level.go:132`), calling [`internal/activities/points.go` lines 18–46](https://github.com/temporal-sa/temporal-demo-wordflow/blob/beb6d7799792b2dc8dadf57750a6d9c7372ed9a3/internal/activities/points.go#L18-L46) (`code -g internal/activities/points.go:18`)

```go
pointsSpent := 0
var pointsRemaining *int
if freeHintCount(state, update.Hint) == 0 {
	price, err := hintPointCost(state.Puzzle, update.Hint)
	if err != nil {
		return game.HintResult{}, err
	}
	updateID := fmt.Sprintf("spend/%s/%s", state.WorkflowID, workflow.GetCurrentUpdateInfo(updateCtx).ID)
	activityCtx := workflow.WithActivityOptions(updateCtx, workflow.ActivityOptions{StartToCloseTimeout: 30 * time.Second})
	var spend SpendPointsResult
	err = workflow.ExecuteActivity(activityCtx, ActivitySpendPoints, SpendPointsActivityInput{
		PlayerWorkflowID: PlayerWorkflowID(state.PlayerID),
		UpdateID:         updateID,
		// ...
	}).Get(updateCtx, &spend)
```

```go
handle, err := temporalClient.UpdateWorkflow(ctx, client.UpdateWorkflowOptions{
	WorkflowID:   input.PlayerWorkflowID,
	UpdateID:     input.UpdateID,
	UpdateName:   workflows.UpdateSpendPoints,
	WaitForStage: client.WorkflowUpdateStageCompleted,
	Args:         []any{input.Spend},
})
// ...
var result workflows.SpendPointsResult
if err := handle.Get(ctx, &result); err != nil {
	var applicationError *temporal.ApplicationError
	if errors.As(err, &applicationError) {
		return workflows.SpendPointsResult{}, temporal.NewNonRetryableApplicationError(
			applicationError.Message(), applicationError.Type(), err,
		)
	}
	return workflows.SpendPointsResult{}, err
}
```

**What you need to understand:**

- A Go Workflow can signal another Workflow but has no API to send it an Update. The level sends the Update from an Activity, and gets the player's answer back as the Activity result.
- The Update ID is built from the level's Workflow ID and the ID of the `use-hint` Update that triggered it. An Activity retry sends the same ID, so the player deducts once and returns the same result.
- When the player rejects the spend, for example `insufficient_points`, the Activity turns that into a non-retryable error. Retrying wouldn't produce points, so the hint fails at once.
- The player checks that the level asking is its active level and that the balance covers the price, then deducts. The level never touches the balance.
- The default Activity Retry Policy applies otherwise, so a network error or a missing Worker on the player's side is retried.

**Docs:** [Request-Response via Updates](https://docs.temporal.io/design-patterns/request-response-via-updates), [Retry Policies](https://docs.temporal.io/encyclopedia/retry-policies)

### Publish scores to a leaderboard that ignores stale ones

**When to use it:** during beat 5.

**Where it is:** [`internal/activities/leaderboard.go` lines 29–36](https://github.com/temporal-sa/temporal-demo-wordflow/blob/beb6d7799792b2dc8dadf57750a6d9c7372ed9a3/internal/activities/leaderboard.go#L29-L36) (`code -g internal/activities/leaderboard.go:29`), received in [`internal/workflows/leaderboard.go` lines 73–91](https://github.com/temporal-sa/temporal-demo-wordflow/blob/beb6d7799792b2dc8dadf57750a6d9c7372ed9a3/internal/workflows/leaderboard.go#L73-L91) (`code -g internal/workflows/leaderboard.go:73`)

```go
_, err = temporalClient.SignalWithStartWorkflow(ctx, workflows.LeaderboardWorkflowID,
	workflows.SignalLeaderboardScore, entry, client.StartWorkflowOptions{
		ID:                       workflows.LeaderboardWorkflowID,
		TaskQueue:                taskQueue,
		WorkflowIDConflictPolicy: enums.WORKFLOW_ID_CONFLICT_POLICY_USE_EXISTING,
		WorkflowIDReusePolicy:    enums.WORKFLOW_ID_REUSE_POLICY_ALLOW_DUPLICATE_FAILED_ONLY,
	}, workflows.LeaderboardWorkflowName, workflows.LeaderboardWorkflowInput{})
return err
```

```go
func upsertLeaderboardEntry(state *LeaderboardState, entry game.LeaderboardEntry) {
	for index := range state.Entries {
		if state.Entries[index].PlayerID != entry.PlayerID {
			continue
		}
		if entry.Version <= state.Entries[index].Version {
			return
		}
		state.Entries[index] = entry
		sortLeaderboard(state)
		return
	}

	state.Entries = append(state.Entries, entry)
	sortLeaderboard(state)
	if len(state.Entries) > leaderboardSize {
		state.Entries = state.Entries[:leaderboardSize]
	}
}
```

**What you need to understand:**

- Signal-With-Start sends the score to `leaderboard/global`, starting it first if it isn't running. Nobody has to create the leaderboard before the first game.
- The player sends its absolute lifetime total, not a delta, with a version equal to its completed-level count. A retried or late Signal carries a version the leaderboard already has, and it's dropped.
- Signals have no built-in deduplication. The version check is the Workflow doing it itself, which is the pattern the docs recommend for Signals.
- The leaderboard keeps only the top 100, so its state stays small however many people play.
- The `/leaderboard` page reads it with a Query every 3 seconds. Each Query is an Action on Temporal Cloud.

**Docs:** [Signal-With-Start](https://docs.temporal.io/develop/go/workflows/message-passing#signal-with-start), [Message handler patterns](https://docs.temporal.io/handling-messages#exactly-once-message-processing)

### Pin each Workflow to its build, and upgrade at Continue-As-New

**When to use it:** during beat 6, or when someone asks how a Workflow that never ends takes new code.

**Where it is:** [`internal/workflows/names.go` lines 70–105](https://github.com/temporal-sa/temporal-demo-wordflow/blob/beb6d7799792b2dc8dadf57750a6d9c7372ed9a3/internal/workflows/names.go#L70-L105) (`code -g internal/workflows/names.go:70`), and the Player's loop in [`internal/workflows/player.go` lines 275–282](https://github.com/temporal-sa/temporal-demo-wordflow/blob/beb6d7799792b2dc8dadf57750a6d9c7372ed9a3/internal/workflows/player.go#L275-L282) and [lines 412–416](https://github.com/temporal-sa/temporal-demo-wordflow/blob/beb6d7799792b2dc8dadf57750a6d9c7372ed9a3/internal/workflows/player.go#L412-L416) (`code -g internal/workflows/player.go:275`)

```go
// Register uses pinned registrations when Worker Versioning is enabled.
func Register(registry worker.Registry, versioned bool) {
	// ...
	registry.RegisterWorkflowWithOptions(PlayerWorkflow, workflow.RegisterOptions{
		Name:               PlayerWorkflowName,
		VersioningBehavior: workflow.VersioningBehaviorPinned,
	})
	// ...
}
```

```go
if leaderboardFuture == nil && shouldContinuePlayer(ctx, state, continueRequested) {
	if err := workflow.Await(ctx, func() bool { return workflow.AllHandlersFinished(ctx) }); err != nil {
		return err
	}
	return workflow.NewContinueAsNewErrorWithOptions(ctx, workflow.ContinueAsNewErrorOptions{
		InitialVersioningBehavior: workflow.ContinueAsNewVersioningBehaviorAutoUpgrade,
	}, PlayerWorkflowName, PlayerWorkflowInput{State: state})
}
```

```go
func shouldContinuePlayer(ctx workflow.Context, state *PlayerState, continueRequested bool) bool {
	return state.ActiveGame == nil && (continueRequested ||
		workflow.GetInfo(ctx).GetContinueAsNewSuggested() ||
		workflow.GetInfo(ctx).GetTargetWorkerDeploymentVersionChanged())
}
```

**What you need to understand:**

- With a Build ID set, every Workflow type is registered as Pinned: a run stays on the Worker Deployment Version it started on. The Lambda Worker always runs versioned; `cmd/worker` runs versioned only when `TEMPORAL_WORKER_BUILD_ID` is set, which the `steady` component does on tmprl-demo.cloud.
- The Player continues as new when the server suggests it (history is getting large), or when the Task Queue's target version has changed. Either way it carries its whole state into the new run.
- `ContinueAsNewVersioningBehaviorAutoUpgrade` starts the new run on the current target version. After its first Workflow Task, the new run is Pinned again, now to the new build.
- The flag is refreshed after each Workflow Task, so an idle player notices a new version at its next Update. Each entity also listens for a `request-version-upgrade` Signal that forces the move; nothing in the repo sends it.
- The Player won't continue while a level is open, because the open level is its child and the Player is waiting on its result. The Level, Catalog, and Leaderboard Workflows check the same flags in their own loops, so an open level moves to the new build at its next guess. Each waits for in-flight handlers before it continues.
- There is no patching or version branching in the Workflow code. Upgrade on Continue-As-New is in Public Preview as an experimental SDK option, and Serverless Workers are in Public Preview.

**Docs:** [Upgrade on Continue-as-New](https://docs.temporal.io/production-deployment/worker-deployments/worker-versioning/upgrade-on-continue-as-new), [Worker Versioning](https://docs.temporal.io/worker-versioning), [Serverless Workers on AWS Lambda (Go)](https://docs.temporal.io/develop/go/workers/serverless-workers/aws-lambda)

### What's demo-only

When someone asks what they'd change for their own system, these are the parts to leave out or replace:

- **The identity model.** [`internal/api/auth.go`](https://github.com/temporal-sa/temporal-demo-wordflow/blob/beb6d7799792b2dc8dadf57750a6d9c7372ed9a3/internal/api/auth.go) derives a deterministic password hash from the username and signs a 30-day session JWT. It has no email verification, account recovery, MFA, or rate limiting. A real app would use an identity provider and keep only the user ID in the Workflow.
- **The Workflow Explorer.** [`internal/explorer/server.go` lines 37–41](https://github.com/temporal-sa/temporal-demo-wordflow/blob/beb6d7799792b2dc8dadf57750a6d9c7372ed9a3/internal/explorer/server.go#L37-L41) uses only List, Describe, and Get History, and sanitizes every payload, so visitors can see the machinery without a Temporal login. It's for the demo audience; operators use the Temporal UI.
- **Seeding the campaign at server start.** [`cmd/server/main.go` lines 35–37](https://github.com/temporal-sa/temporal-demo-wordflow/blob/beb6d7799792b2dc8dadf57750a6d9c7372ed9a3/cmd/server/main.go#L35-L37) starts the built-in campaign every time the web server starts. The conflict policy makes that a no-op once it's running. A real system would start campaigns from an admin tool, as `cmd/campaign` does.
- **The game rules.** Scoring, hints, streaks, and unlock timing are game logic, not Temporal features.

## Reset and tear down

- **Between presentations:** sign up with a new username. Every player is separate, so nothing else needs resetting. The leaderboard keeps earlier players.
- **To reuse a username locally:** terminate its Player Workflow, for example `temporal workflow terminate --workflow-id player/<username>`. The ID reuse policy then lets **Create account** claim it again. Its open level is a child Workflow and closes with it.
- **To wipe everything locally:** stop the dev server and delete `.temporal/`.
- **To stop locally:** `Ctrl-C` the server, the Worker, and the dev server.
- **On tmprl-demo.cloud:** there's nothing to stop. The open Workflows wait for Updates and Signals and schedule no Timers or Activities on their own. Close the leaderboard tab when you're done, because it runs a Query every 3 seconds.

## When something goes wrong

| What you see | Likely cause | Fix |
|---|---|---|
| `make worker` or `make server` exits with `tls: first record does not look like a TLS handshake` | Your default Temporal CLI profile points at Temporal Cloud | Set `TEMPORAL_CONFIG_FILE` to an empty file, as in [Before you start](#before-you-start) |
| `make server` exits with `SESSION_JWT_SECRET is required` | You ran `go run ./cmd/server` directly | Use `make server`, which sets a local secret |
| The page stays on **Loading campaigns** or a button stays on **Submitting…** | No Worker is running; every Query and Update waits for one | Start `make worker` |
| **Reveal one** fails with `not enough points` | The player hasn't won a level yet | Finish level 1 first; it pays at least 10 points |
| **Start campaign** fails with `finish the active level before starting another` | A level is still open | Press **Resume level N** and finish it |
| **Create account** fails with `username is already registered with a different password` | The username is taken | Use a new username |
| The Explorer shows no **Versioning** strip | The Worker is unversioned, which is normal locally | Show beat 6 on tmprl-demo.cloud |

## Keeping this runbook current

The code links point at a fixed commit. When a linked file changes, update its link, line numbers, and snippet together. Before an event, list the linked files that have changed:

```sh
grep -oE 'blob/[0-9a-f]{40}/[^#)]+' RUNBOOK.md | sort -u | while IFS=/ read -r _ sha file; do
  git diff --quiet "$sha" HEAD -- "$file" || echo "changed since ${sha:0:7}: $file"
done
```
