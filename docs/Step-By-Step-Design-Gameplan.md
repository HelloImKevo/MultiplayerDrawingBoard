# Multiplayer Drawing Board POC

This guide walks you through creating a lightweight proof-of-concept multiplayer drawing board from scratch using:

- .NET 7
- ASP.NET Core SignalR
- RxJS
- TypeScript
- VS Code

The goal is speed, not polish. By the end, you will have:

- a .NET 7 backend hosting a SignalR hub
- a small TypeScript frontend that draws on an HTML canvas
- real-time stroke syncing between multiple browser windows
- basic local build and test steps that work well in VS Code

## 1. What You Are Building

The app has two parts:

1. A backend SignalR hub that broadcasts drawing events to everyone on the same board.
2. A frontend canvas app that sends mouse movements as drawing strokes and listens for remote strokes.

For a quick POC, keep the data model simple:

- no database
- no login
- no persistence after refresh
- one shared board room

That gives you the smallest working multiplayer loop.

## 2. Prerequisites

Install these before you start:

- .NET SDK 7.x
- Node.js 20.x or current LTS
- VS Code
- C# Dev Kit extension for VS Code
- ESLint or Prettier extensions if you want formatting help

Verify your tools:

```bash
dotnet --version
node --version
npm --version
```

You want .NET to report a 7.x SDK.

## 3. Create the Project Structure

Open a terminal in VS Code and run:

```bash
mkdir MultiplayerDrawingBoard
cd MultiplayerDrawingBoard

dotnet new sln -n MultiplayerDrawingBoard

dotnet new web -n Server --framework net7.0
npm create vite@latest client -- --template vanilla-ts

dotnet sln add Server/Server.csproj
```

Your folder should now look like this:

```text
MultiplayerDrawingBoard/
  Server/
  client/
  MultiplayerDrawingBoard.sln
```

## 4. Install Frontend Dependencies

Move into the client app and install what you need:

```bash
cd client
npm install
npm install rxjs @microsoft/signalr
cd ..
```

Why these packages:

- `rxjs` gives you a lightweight event/state pipeline.
- `@microsoft/signalr` is the JavaScript SignalR client.

## 5. Backend: Add the Drawing Hub

In the `Server` project, create a `Hubs` folder and add `DrawingHub.cs`.

File: `Server/Hubs/DrawingHub.cs`

```csharp
using Microsoft.AspNetCore.SignalR;

namespace Server.Hubs;

public class DrawingHub : Hub
{
    public async Task JoinBoard(string boardId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, boardId);
        await Clients.Caller.SendAsync("JoinedBoard", boardId);
    }

    public async Task SendStroke(string boardId, DrawingStroke stroke)
    {
        await Clients.OthersInGroup(boardId).SendAsync("ReceiveStroke", stroke);
    }

    public async Task ClearBoard(string boardId)
    {
        await Clients.Group(boardId).SendAsync("BoardCleared");
    }
}

public record DrawingPoint(double X, double Y);

public record DrawingStroke(
    string Color,
    double Width,
    List<DrawingPoint> Points
);
```

This hub does three jobs:

- joins a user to a board group
- broadcasts strokes to everyone else in that board
- broadcasts a clear event

## 6. Backend: Configure SignalR and CORS

Replace the contents of `Server/Program.cs` with this:

```csharp
using Server.Hubs;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddCors(options =>
{
    options.AddPolicy("ClientPolicy", policy =>
    {
        policy
            .WithOrigins("http://localhost:5173")
            .AllowAnyHeader()
            .AllowAnyMethod()
            .AllowCredentials();
    });
});

builder.Services.AddSignalR();

var app = builder.Build();

app.UseHttpsRedirection();
app.UseCors("ClientPolicy");

app.MapGet("/health", () => Results.Ok(new { status = "ok" }));
app.MapHub<DrawingHub>("/hubs/drawing");

app.Run();
```

Why this matters:

- SignalR needs `AddSignalR` and `MapHub`.
- The browser client needs CORS configured to allow the Vite dev server.
- `/health` gives you a quick smoke-test endpoint.

## 7. Frontend: Replace the Default Vite App

You can remove the default demo files or overwrite them.

### 7.1 Replace `client/src/main.ts`

```ts
import './style.css';
import { BehaviorSubject, Subject } from 'rxjs';
import * as signalR from '@microsoft/signalr';

type DrawingPoint = {
  x: number;
  y: number;
};

type DrawingStroke = {
  color: string;
  width: number;
  points: DrawingPoint[];
};

const boardId = 'main-room';
const connectionState$ = new BehaviorSubject<'connecting' | 'connected' | 'disconnected'>('connecting');
const remoteStroke$ = new Subject<DrawingStroke>();
const boardCleared$ = new Subject<void>();

const app = document.querySelector<HTMLDivElement>('#app');

if (!app) {
  throw new Error('App container not found.');
}

app.innerHTML = `
  <div class="page">
    <div class="toolbar">
      <div>
        <h1>Multiplayer Drawing Board</h1>
        <p id="status">Connecting...</p>
      </div>
      <div class="actions">
        <input id="colorPicker" type="color" value="#1d4ed8" />
        <input id="brushSize" type="range" min="1" max="12" value="3" />
        <button id="clearBtn">Clear board</button>
      </div>
    </div>
    <canvas id="board" width="1000" height="700"></canvas>
  </div>
`;

const canvas = document.querySelector<HTMLCanvasElement>('#board');
const status = document.querySelector<HTMLParagraphElement>('#status');
const clearBtn = document.querySelector<HTMLButtonElement>('#clearBtn');
const colorPicker = document.querySelector<HTMLInputElement>('#colorPicker');
const brushSize = document.querySelector<HTMLInputElement>('#brushSize');

if (!canvas || !status || !clearBtn || !colorPicker || !brushSize) {
  throw new Error('UI elements not found.');
}

const context = canvas.getContext('2d');

if (!context) {
  throw new Error('2D canvas context not available.');
}

context.lineCap = 'round';
context.lineJoin = 'round';

const connection = new signalR.HubConnectionBuilder()
  .withUrl('http://localhost:5167/hubs/drawing')
  .withAutomaticReconnect()
  .build();

connection.on('ReceiveStroke', (stroke: DrawingStroke) => {
  remoteStroke$.next(stroke);
});

connection.on('BoardCleared', () => {
  boardCleared$.next();
});

connection.onreconnecting(() => {
  connectionState$.next('connecting');
});

connection.onreconnected(async () => {
  connectionState$.next('connected');
  await connection.invoke('JoinBoard', boardId);
});

connection.onclose(() => {
  connectionState$.next('disconnected');
});

connectionState$.subscribe((state) => {
  const text =
    state === 'connected'
      ? 'Connected'
      : state === 'connecting'
        ? 'Connecting...'
        : 'Disconnected';

  status.textContent = text;
});

function drawStroke(stroke: DrawingStroke) {
  if (stroke.points.length < 2) {
    return;
  }

  context.strokeStyle = stroke.color;
  context.lineWidth = stroke.width;
  context.beginPath();
  context.moveTo(stroke.points[0].x, stroke.points[0].y);

  for (let index = 1; index < stroke.points.length; index += 1) {
    const point = stroke.points[index];
    context.lineTo(point.x, point.y);
  }

  context.stroke();
}

function clearCanvas() {
  context.clearRect(0, 0, canvas.width, canvas.height);
}

remoteStroke$.subscribe(drawStroke);
boardCleared$.subscribe(clearCanvas);

let isDrawing = false;
let currentPoints: DrawingPoint[] = [];

function getCanvasPoint(event: PointerEvent): DrawingPoint {
  const rect = canvas.getBoundingClientRect();
  return {
    x: event.clientX - rect.left,
    y: event.clientY - rect.top,
  };
}

canvas.addEventListener('pointerdown', (event) => {
  isDrawing = true;
  currentPoints = [getCanvasPoint(event)];
});

canvas.addEventListener('pointermove', (event) => {
  if (!isDrawing) {
    return;
  }

  currentPoints.push(getCanvasPoint(event));

  const previewStroke: DrawingStroke = {
    color: colorPicker.value,
    width: Number(brushSize.value),
    points: [...currentPoints],
  };

  clearCanvas();
  drawStroke(previewStroke);
});

async function finishStroke() {
  if (!isDrawing || currentPoints.length < 2) {
    isDrawing = false;
    currentPoints = [];
    return;
  }

  const stroke: DrawingStroke = {
    color: colorPicker.value,
    width: Number(brushSize.value),
    points: [...currentPoints],
  };

  drawStroke(stroke);
  await connection.invoke('SendStroke', boardId, stroke);
  isDrawing = false;
  currentPoints = [];
}

canvas.addEventListener('pointerup', () => {
  void finishStroke();
});

canvas.addEventListener('pointerleave', () => {
  void finishStroke();
});

clearBtn.addEventListener('click', async () => {
  clearCanvas();
  await connection.invoke('ClearBoard', boardId);
});

async function start() {
  await connection.start();
  await connection.invoke('JoinBoard', boardId);
  connectionState$.next('connected');
}

void start();
```

### 7.2 Replace `client/src/style.css`

```css
:root {
  color: #0f172a;
  background: linear-gradient(180deg, #f8fafc 0%, #e2e8f0 100%);
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

* {
  box-sizing: border-box;
}

body {
  margin: 0;
  min-width: 320px;
}

.page {
  max-width: 1100px;
  margin: 0 auto;
  padding: 24px;
}

.toolbar {
  display: flex;
  justify-content: space-between;
  gap: 16px;
  align-items: center;
  margin-bottom: 16px;
  flex-wrap: wrap;
}

.toolbar h1 {
  margin: 0 0 8px;
  font-size: 2rem;
}

.toolbar p {
  margin: 0;
}

.actions {
  display: flex;
  gap: 12px;
  align-items: center;
}

button {
  border: none;
  border-radius: 10px;
  padding: 10px 16px;
  background: #1d4ed8;
  color: white;
  cursor: pointer;
}

canvas {
  width: 100%;
  background: white;
  border-radius: 16px;
  box-shadow: 0 10px 30px rgba(15, 23, 42, 0.12);
  touch-action: none;
}
```

## 8. Important POC Note About Local Drawing

The sample code above is intentionally minimal. It draws remote strokes correctly, but the local preview logic clears the canvas before re-drawing the current in-progress line.

For a real version, you should keep an in-memory array of all strokes and re-render the full board every frame.

For a fast proof of concept, the simplest improvement is:

- keep `const allStrokes: DrawingStroke[] = [];`
- push each completed local or remote stroke into the array
- replace `clearCanvas()` plus single-stroke drawing with a `renderBoard(allStrokes)` function

That change is worth doing if you want the local user to see a stable accumulated drawing.

## 9. Better Frontend Rendering for a Stable Board

If you want the cleaner POC immediately, refactor the drawing section like this.

Replace the rendering-related logic in `main.ts` with this pattern:

```ts
const allStrokes: DrawingStroke[] = [];

function renderBoard() {
  clearCanvas();
  for (const stroke of allStrokes) {
    drawStroke(stroke);
  }
}

remoteStroke$.subscribe((stroke) => {
  allStrokes.push(stroke);
  renderBoard();
});

boardCleared$.subscribe(() => {
  allStrokes.length = 0;
  clearCanvas();
});

async function finishStroke() {
  if (!isDrawing || currentPoints.length < 2) {
    isDrawing = false;
    currentPoints = [];
    return;
  }

  const stroke: DrawingStroke = {
    color: colorPicker.value,
    width: Number(brushSize.value),
    points: [...currentPoints],
  };

  allStrokes.push(stroke);
  renderBoard();
  await connection.invoke('SendStroke', boardId, stroke);
  isDrawing = false;
  currentPoints = [];
}
```

That version is closer to what you actually want.

## 10. Run the Backend in VS Code

Open a VS Code terminal in the `Server` folder and run:

```bash
dotnet restore
dotnet run
```

You should see a localhost URL in the output.

If you want automatic reload while editing:

```bash
dotnet watch run
```

Before wiring the frontend, test the health endpoint in your browser:

```text
http://localhost:5167/health
```

If your port is different, use the port printed by ASP.NET Core.

## 11. Run the Frontend in VS Code

Open a second VS Code terminal in the `client` folder and run:

```bash
npm run dev
```

Vite will print a URL, usually:

```text
http://localhost:5173
```

Open it in the browser.

## 12. Make the SignalR Client URL Match the Server Port

This is the part that usually trips people up.

Your frontend contains this line:

```ts
.withUrl('http://localhost:5167/hubs/drawing')
```

If the backend starts on a different port, update that URL to match the real backend port.

A good next step is to move the base URL into a Vite environment variable later, but hard-coding it is fine for a first POC.

## 13. Smoke-Test Multiplayer Behavior

Use this test flow:

1. Start the backend.
2. Start the frontend.
3. Open the frontend in two browser windows.
4. Draw in window A.
5. Confirm the stroke appears in window B.
6. Click `Clear board`.
7. Confirm both windows clear.

If that works, your POC is functioning.

## 14. Build Checks

Use these commands before you consider the POC stable:

### Backend build

```bash
cd Server
dotnet build
```

### Frontend build

```bash
cd client
npm run build
```

These checks are enough for a first pass.

## 15. Recommended VS Code Workflow

A simple working setup in VS Code:

1. Open the `MultiplayerDrawingBoard` folder in VS Code.
2. Split the terminal into two panes.
3. Run `dotnet watch run` in `Server`.
4. Run `npm run dev` in `client`.
5. Keep the browser dev tools open to watch SignalR and console errors.

Useful files to keep side-by-side:

- `Server/Program.cs`
- `Server/Hubs/DrawingHub.cs`
- `client/src/main.ts`
- `client/src/style.css`

## 16. Common Problems

### CORS error

Symptom:

- browser console shows blocked cross-origin requests

Fix:

- make sure `WithOrigins("http://localhost:5173")` matches the real Vite URL
- make sure `.AllowCredentials()` is present
- restart the backend after editing CORS settings

### SignalR connection fails

Symptom:

- browser console shows negotiation or connection errors

Fix:

- check the hub path is exactly `/hubs/drawing`
- verify the backend port in `.withUrl(...)`
- confirm the backend is running before the frontend connects

### Nothing shows in the second browser

Symptom:

- local drawing works, remote drawing does not

Fix:

- verify `SendStroke` is being invoked
- verify `ReceiveStroke` is registered on the client
- confirm both clients joined the same `boardId`

### Canvas looks glitchy while drawing

Symptom:

- in-progress lines flicker or old lines disappear

Fix:

- move to the `allStrokes` plus `renderBoard()` pattern from section 9

## 17. Quick Enhancements After the POC Works

Once the basic version is working, these are the best next upgrades:

1. Add user nicknames so each person is identifiable.
2. Add multiple board rooms using a room code text box.
3. Persist strokes in memory on the server for late joiners.
4. Save boards to a database.
5. Add touch support improvements for tablets.
6. Add throttling or batching so drawing emits fewer network messages.

## 18. Minimal Architecture Summary

Keep the mental model simple:

- canvas input creates a `DrawingStroke`
- frontend sends the stroke to SignalR
- SignalR hub broadcasts the stroke to the group
- RxJS streams deliver events to the rendering logic
- canvas re-renders the board

That is the full loop.

## 19. If You Want an Even Simpler First Version

If this still feels like too much at once, reduce scope further:

- skip brush size
- skip color picker
- hard-code `boardId = 'main-room'`
- only implement `SendStroke`
- skip `ClearBoard` until the basic draw-sync loop works

That is a better approach than trying to build too much before the first successful demo.

## 20. Definition of Done for This POC

You are done when:

- `dotnet build` passes
- `npm run build` passes
- two browser windows connect successfully
- strokes drawn in one window appear in the other
- clearing the board updates every connected client

At that point, you have a valid proof of concept.
