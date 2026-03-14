# Copilot Workspace Instructions

**SocOps** is a Social Bingo game built with Blazor WebAssembly (.NET 10). Players find people who match icebreaker prompts to mark squares and get 5 in a row.

---

## ✅ MANDATORY: Pre-Commit Checklist

**MUST PASS before any commit:**
- [ ] `dotnet build SocOps/SocOps.csproj` — Compiles with no errors
- [ ] Lint/code style — C# conventions (PascalCase, `private` by default)
- [ ] `dotnet test` — All unit tests passing (or add tests if missing)
- [ ] Components use `[Parameter]` + `EventCallback` pattern
- [ ] Services call `NotifyStateChanged()` on state mutations
- [ ] No unused variables/imports

---

## Quick Start

```bash
cd SocOps
dotnet run          # Dev server on https://localhost:5001
dotnet build        # Compile only
```

---

## Project Structure

```
SocOps/
├── Components/     BingoSquare, BingoBoard, GameScreen, StartScreen, BingoModal
├── Services/       BingoGameService (state+events), BingoLogicService (pure logic)
├── Models/         GameState enum, BingoSquareData, BingoLine
├── Pages/          Home.razor (@page "/")
├── Data/           Questions.cs (24 icebreaker prompts)
└── wwwroot/css/    app.css (custom utilities)

---

## State Management

### BingoGameService (Scoped DI)
Central state manager in [SocOps/Services/BingoGameService.cs](SocOps/Services/BingoGameService.cs):
- Maintains: `CurrentGameState`, `Board`, `WinningLine`, `ShowBingoModal`
- Publishes: `OnStateChanged` event when state mutates
- Methods: `StartGame()`, `HandleSquareClick()`, `ResetGame()`, `DismissModal()`
- Persists: Board state to localStorage

**Pattern**: When state changes, call `NotifyStateChanged()` → triggers `StateHasChanged()` in subscribed components.

### BingoLogicService (Static)
Pure, testable logic in [SocOps/Services/BingoLogicService.cs](SocOps/Services/BingoLogicService.cs):
- `GenerateBoard()` — Randomizes 24 questions, pins FREE SPACE at center
- `ToggleSquare(squareId, board)` — Immutable board mutation
- `CheckBingo(board)` — Detects 5-in-a-row; returns `BingoLine?`

### Game State Flow
```
Start → [StartGame()] → Playing → [5 bingo detected] → Bingo (modal) → Playing → [Reset] → Start
```

---

## Models & Data

**GameState** enum: `Start`, `Playing`, `Bingo`  
**BingoSquareData**: `Id`, `Text`, `IsMarked`, `IsFreeSpace`  
**BingoLine**: `Type` (row/column/diagonal), `Index`, `Squares[]`

Questions stored in [SocOps/Data/Questions.cs](SocOps/Data/Questions.cs) (24 icebreaker prompts). Keep concise (2–5 words), inclusive, mixed topics.

---

## Component Patterns

### Parameter + EventCallback
```razor
@* Child *@
@code {
    [Parameter] public BingoSquareData Square { get; set; }
    [Parameter] public EventCallback<int> OnClick { get; set; }
    private async Task HandleClick() => await OnClick.InvokeAsync(Square.Id);
}

@* Parent *@
<BingoSquare Square="square" OnClick="@HandleSquareClick" />
```

### Conditional Rendering
```razor
@if (GameService.CurrentGameState == GameState.Start)
    <StartScreen OnStart="@OnStartClicked" />
else
    <GameScreen Board="@GameService.Board" OnReset="@OnReset" />
    @if (GameService.ShowBingoModal)
        <BingoModal WinningLine="@GameService.WinningLine" OnDismiss="@OnDismissModal" />
```

### Disposal
Always unsubscribe in `Dispose()`:
```csharp
@implements IDisposable

protected override async Task OnInitializedAsync()
{
    GameService.OnStateChanged += StateHasChanged;
}

void IDisposable.Dispose() => GameService.OnStateChanged -= StateHasChanged;
```

---

## Frontend Design & Styling

Use custom CSS utilities in [SocOps/wwwroot/css/app.css](SocOps/wwwroot/css/app.css) (no external frameworks). See [css-utilities.instructions.md](.github/instructions/css-utilities.instructions.md) for utilities reference.

When designing, follow the **frontend-design skill**:
- Use distinctive fonts (avoid generic Arial, Inter, Roboto)
- Commit to cohesive color themes with CSS variables
- Leverage animations for high-impact moments
- Layer gradients/patterns for atmosphere
- Avoid "AI slop" — make unexpected, context-specific choices

---

## Extension Points

**Add game variant**: Create `Models/{VariantState}.cs`, add logic to `BingoLogicService.cs`, create components, wire into `Home.razor`

**Add localStorage persistence**: Use `IJSRuntime` (already in place)
```csharp
await JS.InvokeVoidAsync("localStorage.setItem", "gameKey", JsonSerializer.Serialize(state));
var json = await JS.InvokeAsync<string>("localStorage.getItem", "gameKey");
```

**Add CSS animations**: Define keyframes in `app.css`, apply via utility classes
```css
@keyframes slideIn { from { transform: translateY(-100%); opacity: 0; } to { transform: translateY(0); opacity: 1; } }
.animate-slide-in { animation: slideIn 0.5s ease-out; }
```

---

## Common Issues & Solutions

| Issue | Solution |
|-------|----------|
| Component doesn't update | Ensure service calls `NotifyStateChanged()` |
| CSS utility not applying | Check spelling in [app.css](SocOps/wwwroot/css/app.css); add if missing |
| localStorage not persisting | Verify `IJSRuntime` injected; check browser dev tools |
| Event callback not invoked | Confirm parent has `@on*` directive; child uses `await EventCallback.InvokeAsync()` |
| Build fails | Ensure working directory is `SocOps/` |

---

## Agent Productivity Tips

✅ **High-value tasks**
- Add features to `BingoLogicService.cs` (pure logic, highly testable)
- Add questions to [SocOps/Data/Questions.cs](SocOps/Data/Questions.cs)
- Create components in `Components/` following cascade + EventCallback
- Add CSS utilities to [SocOps/wwwroot/css/app.css](SocOps/wwwroot/css/app.css)
- Update `BingoGameService.cs` for new states/persistence

✅ **Before design work**: Review **frontend-design skill**; provide context (users, event, mood)

✅ **Before extending services**: Logic in `BingoLogicService` (pure) or `BingoGameService` (stateful)? Ensure `NotifyStateChanged()` is called.

⚠️ **Avoid**
- External CSS frameworks (maintain utility-first)
- Business logic in components (push to services)
- Forgetting `IDisposable` to unsubscribe from events
- Mutating nested objects (recreate collections)

---

## Key Files

| File | Purpose |
|------|---------|
| [Program.cs](SocOps/Program.cs) | DI configuration, builder setup |
| [App.razor](SocOps/App.razor) | Router and global layout |
| [SocOps/Pages/Home.razor](SocOps/Pages/Home.razor) | Main page, game orchestrator |
| [SocOps/Services/BingoGameService.cs](SocOps/Services/BingoGameService.cs) | State management + event publishing |
| [SocOps/Services/BingoLogicService.cs](SocOps/Services/BingoLogicService.cs) | Game logic (pure, testable) |
| [SocOps/Models/GameState.cs](SocOps/Models/GameState.cs) | State enum |
| [SocOps/Components/StartScreen.razor](SocOps/Components/StartScreen.razor) | Welcome screen |
| [SocOps/Components/GameScreen.razor](SocOps/Components/GameScreen.razor) | Main game board |
| [SocOps/Data/Questions.cs](SocOps/Data/Questions.cs) | Icebreaker prompts |
| [SocOps/wwwroot/css/app.css](SocOps/wwwroot/css/app.css) | CSS utilities |
