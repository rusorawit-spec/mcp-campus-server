# mcp-campus-server
import argparse
import logging
from typing import Any
from mcp.server.fastmcp import FastMCP

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("smart-campus-mcp")

mcp = FastMCP("Smart Campus Assistant", json_response=True)

CAMPUS_ROOMS: dict[str, dict[str, Any]] = {
    "BLD101": {
        "room_code": "BLD101",
        "building": "Building 1",
        "capacity": 35,
        "equipment": ["projector", "whiteboard", "lecture_pc"],
        "available_slots": ["09:00-11:00", "13:00-15:00"],
    },
    "LAB204": {
        "room_code": "LAB204",
        "building": "Computing Wing",
        "capacity": 24,
        "equipment": ["dual_monitors", "projector", "linux_workstations"],
        "available_slots": ["10:00-12:00", "15:00-17:00"],
    },
    "LIB302": {
        "room_code": "LIB302",
        "building": "Central Library",
        "capacity": 12,
        "equipment": ["smart_tv", "whiteboard"],
        "available_slots": ["08:00-10:00", "11:00-13:00", "14:00-16:00"],
    },
}


def _validate_non_empty_str(value: Any, field_name: str) -> str:
    if not isinstance(value, str) or not value.strip():
        raise ValueError(f"Invalid '{field_name}': expected a non-empty string.")
    return value.strip()


def _validate_positive_int(value: Any, field_name: str) -> int:
    if isinstance(value, bool) or not isinstance(value, int) or value <= 0:
        raise ValueError(f"Invalid '{field_name}': expected a positive integer.")
    return value


# --- RESOURCES (2) ---

@mcp.resource("campus://rooms")
def get_all_rooms() -> dict[str, Any]:
    """Return the full catalogue of campus rooms."""
    try:
        return {
            "total_rooms": len(CAMPUS_ROOMS),
            "rooms": list(CAMPUS_ROOMS.values()),
        }
    except Exception as exc:
        logger.exception("Error retrieving campus rooms catalogue")
        return {"error": f"Failed to load campus rooms: {exc}"}


@mcp.resource("campus://rooms/{room_code}")
def get_room_detail(room_code: str) -> dict[str, Any]:
    """Return details for a specific campus room by room_code."""
    try:
        code = _validate_non_empty_str(room_code, "room_code").upper()
        if code not in CAMPUS_ROOMS:
            return {"error": f"Room '{code}' was not found.", "code": "ROOM_NOT_FOUND"}
        return CAMPUS_ROOMS[code]
    except ValueError as exc:
        return {"error": str(exc), "code": "INVALID_PARAMETER"}
    except Exception as exc:
        logger.exception("Unexpected error reading room detail")
        return {"error": f"Internal error reading room: {exc}", "code": "INTERNAL_ERROR"}


# --- TOOLS (2) ---

@mcp.tool()
def search_rooms(min_capacity: int = 1, equipment: str | None = None) -> dict[str, Any]:
    """Search campus rooms by minimum seating capacity and optional equipment."""
    try:
        cap = _validate_positive_int(min_capacity, "min_capacity")
        eq_filter: str | None = None
        if equipment is not None:
            eq_filter = _validate_non_empty_str(equipment, "equipment").lower()

        matches: list[dict[str, Any]] = []
        for room in CAMPUS_ROOMS.values():
            if room["capacity"] < cap:
                continue
            if eq_filter and eq_filter not in [e.lower() for e in room["equipment"]]:
                continue
            matches.append(room)

        return {
            "query": {"min_capacity": cap, "equipment": eq_filter},
            "count": len(matches),
            "results": matches,
        }
    except ValueError as exc:
        return {"error": str(exc), "code": "INVALID_ARGUMENT"}
    except Exception as exc:
        logger.exception("Error executing search_rooms")
        return {"error": f"Failed to search rooms: {exc}", "code": "EXECUTION_ERROR"}


@mcp.tool()
def check_room_availability(room_code: str, time_slot: str) -> dict[str, Any]:
    """Check whether a room is available during a specific time slot (e.g. '09:00-11:00')."""
    try:
        code = _validate_non_empty_str(room_code, "room_code").upper()
        slot = _validate_non_empty_str(time_slot, "time_slot")

        if code not in CAMPUS_ROOMS:
            return {"error": f"Room '{code}' does not exist.", "code": "ROOM_NOT_FOUND"}

        room = CAMPUS_ROOMS[code]
        available = slot in room["available_slots"]
        return {
            "room_code": code,
            "building": room["building"],
            "time_slot": slot,
            "available": available,
            "available_slots": room["available_slots"],
        }
    except ValueError as exc:
        return {"error": str(exc), "code": "INVALID_ARGUMENT"}
    except Exception as exc:
        logger.exception("Error checking room availability")
        return {"error": f"Failed to check availability: {exc}", "code": "EXECUTION_ERROR"}


# --- PROMPTS (2) ---

@mcp.prompt()
def meeting_planner_prompt(attendees: int, purpose: str, preferred_slot: str = "09:00-11:00") -> str:
    """Create a structured prompt to plan a campus meeting and select a suitable room."""
    try:
        count = _validate_positive_int(attendees, "attendees")
        goal = _validate_non_empty_str(purpose, "purpose")
        slot = _validate_non_empty_str(preferred_slot, "preferred_slot")

        return (
            f"You are a Smart Campus Assistant. Plan a session for {count} attendees "
            f"for the purpose of '{goal}' during time slot '{slot}'.\n"
            f"1. Read 'campus://rooms' to inspect available spaces.\n"
            f"2. Call 'search_rooms' with min_capacity={count}.\n"
            f"3. Call 'check_room_availability' for candidate rooms and '{slot}'.\n"
            f"4. Recommend the best room with clear reasoning."
        )
    except ValueError as exc:
        return f"Prompt parameter error: {exc}"


@mcp.prompt()
def study_session_prompt(subject: str, group_size: int) -> str:
    """Create a structured prompt to organise a group study session on campus."""
    try:
        topic = _validate_non_empty_str(subject, "subject")
        size = _validate_positive_int(group_size, "group_size")

        return (
            f"You are a Smart Campus Assistant helping a study group of {size} students "
            f"prepare for '{topic}'.\n"
            f"1. Use 'search_rooms' with min_capacity={size}.\n"
            f"2. Check 'campus://rooms/{{room_code}}' for equipment details.\n"
            f"3. Propose a room option and a concise 60-minute study agenda."
        )
    except ValueError as exc:
        return f"Prompt parameter error: {exc}"


def main() -> None:
    parser = argparse.ArgumentParser(description="Run Smart Campus MCP Server")
    parser.add_argument(
        "--transport",
        choices=["stdio", "streamable-http"],
        default="stdio",
        help="Transport protocol to use ('stdio' or 'streamable-http')",
    )
    parser.add_argument("--host", default="127.0.0.1", help="HTTP host")
    parser.add_argument("--port", type=int, default=8000, help="HTTP port")
    args = parser.parse_args()

    if args.transport == "streamable-http":
        mcp.settings.host = args.host
        mcp.settings.port = args.port
        mcp.run(transport="streamable-http")
    else:
        mcp.run(transport="stdio")


if __name__ == "__main__":
    main()
