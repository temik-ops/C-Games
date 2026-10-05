using System;
using System.Collections.Generic;
using System.Threading;

namespace DotRunner
{
    class Gremlin
    {
        public int X, Y, PX, PY, DX, DY;
        public int StartX, StartY;
        public int Style;
        public int Wait;
        public ConsoleColor Color;
    }

    class Program
    {
        static readonly string[] Layout =
        {
            "###################",
            "#........#........#",
            "#o##.###.#.###.##o#",
            "#.................#",
            "#.##.#.#####.#.##.#",
            "#....#...#...#....#",
            "####.###.#.###.####",
            "#.......   .......#",
            "####.###.#.###.####",
            "#....#...#...#....#",
            "#.##.#.#####.#.##.#",
            "#o...............o#",
            "###################",
        };

        static int W, H;
        static char[,] g;
        static int px, py, dx, dy, ndx, ndy;
        static int score, lives, pelletsLeft, powerTimer, tick;
        static List<Gremlin> gremlins = new List<Gremlin>();
        static Random rng = new Random();

        static readonly int[] DirX = { 0, 0, -1, 1 };
        static readonly int[] DirY = { -1, 1, 0, 0 };

        static bool Free(int x, int y)
        {
            return x >= 0 && y >= 0 && x < W && y < H && g[y, x] != '#';
        }

        static void LoadLevel()
        {
            H = Layout.Length;
            W = Layout[0].Length;
            g = new char[H, W];
            pelletsLeft = 0;
            for (int y = 0; y < H; y++)
                for (int x = 0; x < W; x++)
                {
                    g[y, x] = Layout[y][x];
                    if (g[y, x] == '.' || g[y, x] == 'o') pelletsLeft++;
                }
        }

        static void ResetPositions()
        {
            px = 9; py = 11;
            dx = 0; dy = 0; ndx = 0; ndy = 0;
            powerTimer = 0;
            foreach (var gr in gremlins)
            {
                gr.X = gr.StartX; gr.Y = gr.StartY;
                gr.PX = gr.X; gr.PY = gr.Y;
                gr.DX = 0; gr.DY = 0;
                gr.Wait = 8 + gr.Style * 6;
            }
        }

        static void Main()
        {
            Console.CursorVisible = false;
            Console.Clear();
            ShowTitle();

            LoadLevel();
            gremlins.Add(new Gremlin { StartX = 8, StartY = 7, Style = 0, Color = ConsoleColor.Red });
            gremlins.Add(new Gremlin { StartX = 9, StartY = 7, Style = 1, Color = ConsoleColor.Magenta });
            gremlins.Add(new Gremlin { StartX = 10, StartY = 7, Style = 2, Color = ConsoleColor.Cyan });
            gremlins.Add(new Gremlin { StartX = 7, StartY = 7, Style = 3, Color = ConsoleColor.Green });

            score = 0;
            lives = 3;
            ResetPositions();

            string endMessage = null;

            while (true)
            {
                bool quit = false;
                while (Console.KeyAvailable)
                {
                    var k = Console.ReadKey(true).Key;
                    if (k == ConsoleKey.UpArrow || k == ConsoleKey.W) { ndx = 0; ndy = -1; }
                    else if (k == ConsoleKey.DownArrow || k == ConsoleKey.S) { ndx = 0; ndy = 1; }
                    else if (k == ConsoleKey.LeftArrow || k == ConsoleKey.A) { ndx = -1; ndy = 0; }
                    else if (k == ConsoleKey.RightArrow || k == ConsoleKey.D) { ndx = 1; ndy = 0; }
                    else if (k == ConsoleKey.Escape) quit = true;
                }
                if (quit) { endMessage = "You quit."; break; }

                tick++;
                int oldPx = px, oldPy = py;
                foreach (var gr in gremlins) { gr.PX = gr.X; gr.PY = gr.Y; }

                if (Free(px + ndx, py + ndy)) { dx = ndx; dy = ndy; }
                if (Free(px + dx, py + dy)) { px += dx; py += dy; }

                char c = g[py, px];
                if (c == '.') { g[py, px] = ' '; score += 10; pelletsLeft--; }
                else if (c == 'o') { g[py, px] = ' '; score += 50; pelletsLeft--; powerTimer = 60; }

                if (pelletsLeft <= 0) { Render(); endMessage = "YOU CLEARED THE MAZE! YOU WIN!"; break; }

                bool died = CheckCollisions(oldPx, oldPy);

                if (!died)
                {
                    bool scared = powerTimer > 0;
                    foreach (var gr in gremlins)
                    {
                        if (gr.Wait > 0) { gr.Wait--; continue; }
                        if (scared ? (tick % 2 == 0) : (tick % 4 != 0))
                            MoveGremlin(gr, scared);
                    }
                    died = CheckCollisions(oldPx, oldPy);
                }

                if (powerTimer > 0) powerTimer--;

                Render();

                if (died)
                {
                    lives--;
                    if (lives <= 0) { endMessage = "GAME OVER. The gremlins got you."; break; }
                    Thread.Sleep(900);
                    ResetPositions();
                    while (Console.KeyAvailable) Console.ReadKey(true);
                }

                Thread.Sleep(110);
            }

            Console.SetCursorPosition(0, H + 4);
            Console.ResetColor();
            Console.CursorVisible = true;
            Console.WriteLine();
            Console.WriteLine(endMessage);
            Console.WriteLine("Final score: " + score);
        }

        static bool CheckCollisions(int oldPx, int oldPy)
        {
            foreach (var gr in gremlins)
            {
                if (gr.Wait > 0) continue;
                bool same = gr.X == px && gr.Y == py;
                bool swapped = gr.X == oldPx && gr.Y == oldPy && gr.PX == px && gr.PY == py;
                if (!same && !swapped) continue;

                if (powerTimer > 0)
                {
                    score += 200;
                    gr.X = gr.StartX; gr.Y = gr.StartY;
                    gr.PX = gr.X; gr.PY = gr.Y;
                    gr.DX = 0; gr.DY = 0;
                    gr.Wait = 15;
                }
                else
                {
                    return true;
                }
            }
            return false;
        }

        static void MoveGremlin(Gremlin gr, bool scared)
        {
            var options = new List<int>();
            for (int i = 0; i < 4; i++)
            {
                int nx = gr.X + DirX[i], ny = gr.Y + DirY[i];
                if (!Free(nx, ny)) continue;
                if (nx == gr.X - gr.DX && ny == gr.Y - gr.DY && (gr.DX != 0 || gr.DY != 0)) continue;
                options.Add(i);
            }
            if (options.Count == 0)
            {
                for (int i = 0; i < 4; i++)
                    if (Free(gr.X + DirX[i], gr.Y + DirY[i])) options.Add(i);
            }
            if (options.Count == 0) return;

            int choice;
            if (scared || (gr.Style == 3 && rng.NextDouble() < 0.5) || (gr.Style == 2 && rng.NextDouble() < 0.35))
            {
                choice = options[rng.Next(options.Count)];
            }
            else
            {
                int tx = px, ty = py;
                if (gr.Style == 1) { tx = px + dx * 4; ty = py + dy * 4; }
                choice = options[0];
                int best = int.MaxValue;
                foreach (int i in options)
                {
                    int nx = gr.X + DirX[i], ny = gr.Y + DirY[i];
                    int d = (nx - tx) * (nx - tx) + (ny - ty) * (ny - ty);
                    if (d < best) { best = d; choice = i; }
                }
            }

            gr.DX = DirX[choice];
            gr.DY = DirY[choice];
            gr.X += gr.DX;
            gr.Y += gr.DY;
        }

        static void Render()
        {
            Console.SetCursorPosition(0, 0);
            Console.ResetColor();
            Console.WriteLine("  DOT RUNNER".PadRight(W * 2));
            Console.WriteLine();

            bool blink = powerTimer > 0 && powerTimer < 20 && tick % 2 == 0;

            for (int y = 0; y < H; y++)
            {
                for (int x = 0; x < W; x++)
                {
                    string cell;
                    ConsoleColor col;

                    Gremlin here = null;
                    foreach (var gr in gremlins)
                        if (gr.X == x && gr.Y == y) { here = gr; break; }

                    if (here != null)
                    {
                        cell = " M";
                        if (powerTimer > 0)
                        {
                            cell = " w";
                            col = blink ? ConsoleColor.White : ConsoleColor.Blue;
                        }
                        else col = here.Color;
                    }
                    else if (x == px && y == py)
                    {
                        cell = " @";
                        col = ConsoleColor.Yellow;
                    }
                    else
                    {
                        switch (g[y, x])
                        {
                            case '#': cell = "##"; col = ConsoleColor.DarkBlue; break;
                            case '.': cell = " ."; col = ConsoleColor.Gray; break;
                            case 'o': cell = " O"; col = ConsoleColor.White; break;
                            default: cell = "  "; col = ConsoleColor.Black; break;
                        }
                    }
                    Console.ForegroundColor = col;
                    Console.Write(cell);
                }
                Console.WriteLine();
            }

            Console.ResetColor();
            Console.WriteLine();
            string hud = " SCORE: " + score + "   LIVES: " + new string('@', Math.Max(lives, 0)) +
                         "   PELLETS LEFT: " + pelletsLeft + (powerTimer > 0 ? "   POWER!" : "");
            Console.WriteLine(hud.PadRight(W * 2));
            Console.WriteLine(" Arrows/WASD: move   Esc: quit".PadRight(W * 2));
        }

        static void ShowTitle()
        {
            Console.WriteLine();
            Console.WriteLine("   ===== P A C   M A N =====");
            Console.WriteLine();
            Console.WriteLine("   You are the @. Eat every pellet in the maze.");
            Console.WriteLine("   Avoid the gremlins (M) - they hunt you in different ways.");
            Console.WriteLine("   Grab a big O pellet to scare them (w) and eat them for bonus points.");
            Console.WriteLine();
            Console.WriteLine("   Arrow keys or WASD ... move");
            Console.WriteLine("   Esc .................. quit");
            Console.WriteLine();
            Console.WriteLine("   Press any key to start...");
            Console.ReadKey(true);
            Console.Clear();
        }
    }
}
