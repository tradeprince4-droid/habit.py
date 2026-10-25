#!/usr/bin/env python3
"""Habit Tracker CLI: track daily habits with streaks, stored in SQLite."""
import argparse
import sqlite3
import sys
from datetime import date, timedelta
from pathlib import Path

DB_PATH = Path.home() / ".habits.db"


def connect():
    con = sqlite3.connect(DB_PATH)
    con.execute("PRAGMA foreign_keys = ON")
    con.execute(
        "CREATE TABLE IF NOT EXISTS habits ("
        "id INTEGER PRIMARY KEY, name TEXT UNIQUE NOT NULL)"
    )
    con.execute(
        "CREATE TABLE IF NOT EXISTS checkins ("
        "habit_id INTEGER NOT NULL, day TEXT NOT NULL, "
        "PRIMARY KEY (habit_id, day), "
        "FOREIGN KEY (habit_id) REFERENCES habits(id) ON DELETE CASCADE)"
    )
    return con


def get_habit(con, name):
    row = con.execute("SELECT id FROM habits WHERE name = ?", (name,)).fetchone()
    if not row:
        sys.exit(f"No habit named '{name}'. Add it with: habit.py add \"{name}\"")
    return row[0]


def days_for(con, habit_id):
    rows = con.execute("SELECT day FROM checkins WHERE habit_id = ?", (habit_id,))
    return {date.fromisoformat(r[0]) for r in rows}


def current_streak(days):
    today = date.today()
    cursor = today if today in days else today - timedelta(days=1)
    streak = 0
    while cursor in days:
        streak += 1
        cursor -= timedelta(days=1)
    return streak


def best_streak(days):
    best = run = 0
    prev = None
    for d in sorted(days):
        run = run + 1 if prev and d - prev == timedelta(days=1) else 1
        best = max(best, run)
        prev = d
    return best


def cmd_add(args):
    con = connect()
    try:
        con.execute("INSERT INTO habits (name) VALUES (?)", (args.name,))
        con.commit()
        print(f"Added '{args.name}'.")
    except sqlite3.IntegrityError:
        sys.exit(f"'{args.name}' already exists.")


def cmd_done(args):
    con = connect()
    hid = get_habit(con, args.name)
    day = date.fromisoformat(args.date) if args.date else date.today()
    con.execute("INSERT OR IGNORE INTO checkins VALUES (?, ?)", (hid, day.isoformat()))
    con.commit()
    streak = current_streak(days_for(con, hid))
    print(f"Checked off '{args.name}' for {day}. Current streak: {streak}")


def cmd_list(_):
    con = connect()
    habits = con.execute("SELECT id, name FROM habits ORDER BY name").fetchall()
    if not habits:
        return print("No habits yet. Add one with: habit.py add \"Read\"")
    today = date.today()
    for hid, name in habits:
        days = days_for(con, hid)
        mark = "x" if today in days else " "
        print(f"[{mark}] {name:<20} streak: {current_streak(days)}")


def cmd_stats(args):
    con = connect()
    hid = get_habit(con, args.name)
    days = days_for(con, hid)
    print(f"{args.name}")
    print(f"  Total check-ins : {len(days)}")
    print(f"  Current streak  : {current_streak(days)}")
    print(f"  Best streak     : {best_streak(days)}")
    last30 = sum(1 for d in days if (date.today() - d).days < 30)
    print(f"  Last 30 days    : {last30}/30 ({last30 * 100 // 30}%)")


def cmd_rm(args):
    con = connect()
    hid = get_habit(con, args.name)
    con.execute("DELETE FROM habits WHERE id = ?", (hid,))
    con.commit()
    print(f"Removed '{args.name}'.")


def main():
    p = argparse.ArgumentParser(description="Track daily habits from the terminal.")
    sub = p.add_subparsers(dest="cmd", required=True)

    a = sub.add_parser("add", help="add a new habit")
    a.add_argument("name")
    a.set_defaults(func=cmd_add)

    d = sub.add_parser("done", help="check off a habit")
    d.add_argument("name")
    d.add_argument("--date", help="YYYY-MM-DD (default: today)")
    d.set_defaults(func=cmd_done)

    sub.add_parser("list", help="show today's status").set_defaults(func=cmd_list)

    s = sub.add_parser("stats", help="show stats for a habit")
    s.add_argument("name")
    s.set_defaults(func=cmd_stats)

    r = sub.add_parser("rm", help="delete a habit")
    r.add_argument("name")
    r.set_defaults(func=cmd_rm)

    args = p.parse_args()
    args.func(args)


if __name__ == "__main__":
    main()
    
