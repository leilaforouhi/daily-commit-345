import argparse
import re
from pathlib import Path


LINK_PATTERN = re.compile(r"!?\[[^\]]*\]\(([^)]+)\)")


def extract_local_links(markdown_file):
    path = Path(markdown_file)

    if not path.is_file():
        raise FileNotFoundError(f"Markdown file not found: {path}")

    text = path.read_text(encoding="utf-8")
    links = []

    for match in LINK_PATTERN.finditer(text):
        target = match.group(1).strip()

        if not target or target.startswith(
            ("http://", "https://", "mailto:", "#", "data:")
        ):
            continue

        target = target.split("#", 1)[0].split("?", 1)[0]

        if target:
            links.append({
                "target": target,
                "line": text.count("\n", 0, match.start()) + 1,
            })

    return links


def check_links(markdown_file):
    source = Path(markdown_file).resolve()
    links = extract_local_links(source)
    results = []

    for link in links:
        target_path = (source.parent / link["target"]).resolve()

        results.append({
            "target": link["target"],
            "line": link["line"],
            "exists": target_path.exists(),
        })

    return results


if __name__ == "__main__":
    parser = argparse.ArgumentParser(
        description="Check local links in a Markdown document."
    )
    parser.add_argument("file", help="Path to the Markdown file")
    args = parser.parse_args()

    try:
        results = check_links(args.file)

        print("\nMarkdown Link Report")
        print("-" * 40)

        if not results:
            print("No local links found.")

        for result in results:
            status = "OK" if result["exists"] else "BROKEN"

            print(
                f"[{status}] Line {result['line']}: "
                f"{result['target']}"
            )

        broken_count = sum(not item["exists"] for item in results)
        print(f"\nTotal links: {len(results)}")
        print(f"Broken links: {broken_count}")

    except (OSError, ValueError) as error:
        print(f"Error: {error}")
