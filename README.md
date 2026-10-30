# Terra Invoice Extractor

Extract vendor details, dates, line items, and totals from invoice images with **GPT-6.1 Terra**, then validate the arithmetic before anything reaches your accounting system.

This project prepares invoice entries for review. A person must approve every result before it is posted, paid, or used for bookkeeping.

[View GPT-6.1 Terra on CometAPI](https://www.cometapi.com/models/openai/gpt-6-1-terra/?utm_source=github&utm_medium=distribution&utm_campaign=bom&utm_content=gpt-6-1-terra)

## Features

- PNG and JPEG invoice input
- Structured JSON output
- Line-item and total validation
- Automatic retry with higher reasoning effort when checks fail
- Human-review flags for unresolved discrepancies

## Setup

Requires Python 3.10 or later.

Install the OpenAI SDK:

```bash
python -m pip install --upgrade openai
```

Set your CometAPI key:

```bash
export COMETAPI_KEY="your_key_here"
```

Keep your API key and invoice files out of source control.

## Run

Place invoice images in an `invoices/` directory:

```text
invoices/
├── invoice-001.png
└── invoice-002.jpg
```

Run the extractor:

```bash
python extract.py invoices/
```

The script writes `results.json` and prints `OK` or `NEEDS REVIEW` for each invoice.

## Script

Save the following as `extract.py`:

```python
import argparse
import base64
import json
import os
from pathlib import Path

from openai import OpenAI

MODEL = "gpt-6.1-terra"
SUPPORTED_EXTENSIONS = {".png", ".jpg", ".jpeg"}

api_key = os.environ.get("COMETAPI_KEY")
if not api_key:
    raise RuntimeError("Set COMETAPI_KEY before running this script.")

client = OpenAI(
    base_url="https://api.cometapi.com/v1",
    api_key=api_key,
    timeout=180.0,
)

LINE_SCHEMA = {
    "type": "object",
    "properties": {
        "description": {"type": "string"},
        "unit_price": {"type": "number"},
        "quantity": {"type": "number"},
        "amount": {"type": "number"},
    },
    "required": [
        "description",
        "unit_price",
        "quantity",
        "amount",
    ],
    "additionalProperties": False,
}

INVOICE_SCHEMA = {
    "type": "object",
    "properties": {
        "vendor": {"type": "string"},
        "invoice_number": {"type": "string"},
        "invoice_date": {"type": "string"},
        "currency": {"type": "string"},
        "lines": {
            "type": "array",
            "items": LINE_SCHEMA,
        },
        "subtotal": {"type": "number"},
        "tax": {"type": "number"},
        "total": {"type": "number"},
    },
    "required": [
        "vendor",
        "invoice_number",
        "invoice_date",
        "currency",
        "lines",
        "subtotal",
        "tax",
        "total",
    ],
    "additionalProperties": False,
}


def extract_invoice(path: Path, effort: str = "medium") -> dict:
    if path.suffix.lower() == ".png":
        mime_type = "image/png"
    else:
        mime_type = "image/jpeg"

    encoded_image = base64.b64encode(
        path.read_bytes()
    ).decode("ascii")

    response = client.responses.create(
        model=MODEL,
        reasoning={"effort": effort},
        input=[
            {
                "role": "user",
                "content": [
                    {
                        "type": "input_text",
                        "text": (
                            "Extract the invoice data exactly as printed. "
                            "Do not guess missing values. Use an empty "
                            "string only when a text field is unreadable."
                        ),
                    },
                    {
                        "type": "input_image",
                        "image_url": (
                            f"data:{mime_type};base64,{encoded_image}"
                        ),
                    },
                ],
            }
        ],
        text={
            "format": {
                "type": "json_schema",
                "name": "invoice",
                "schema": INVOICE_SCHEMA,
                "strict": True,
            }
        },
    )

    if response.status != "completed":
        raise RuntimeError(
            f"Response did not complete: {response.status}"
        )

    if not response.output_text:
        raise RuntimeError("The model returned no structured output.")

    return json.loads(response.output_text)


def validate_invoice(invoice: dict) -> list[str]:
    flags = []

    line_total = round(
        sum(line["amount"] for line in invoice["lines"]),
        2,
    )

    if abs(line_total - invoice["subtotal"]) > 0.01:
        flags.append("line items do not equal subtotal")

    calculated_total = round(
        invoice["subtotal"] + invoice["tax"],
        2,
    )

    if abs(calculated_total - invoice["total"]) > 0.01:
        flags.append("subtotal plus tax does not equal total")

    return flags


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("folder", type=Path)
    args = parser.parse_args()

    if not args.folder.is_dir():
        raise SystemExit(f"Folder not found: {args.folder}")

    files = sorted(
        path
        for path in args.folder.iterdir()
        if path.is_file()
        and path.suffix.lower() in SUPPORTED_EXTENSIONS
    )

    if not files:
        raise SystemExit(
            "No PNG or JPEG files found in the supplied folder."
        )

    results = {}

    for path in files:
        try:
            invoice = extract_invoice(path, effort="medium")
            flags = validate_invoice(invoice)

            if flags:
                invoice = extract_invoice(path, effort="high")
                flags = validate_invoice(invoice)

            results[path.name] = {
                "invoice": invoice,
                "flags": flags,
            }

            status = "NEEDS REVIEW" if flags else "OK"
            print(f"{path.name}: {status}")
            if flags:
                for flag in flags:
                    print(f"  - {flag}")

        except Exception as exc:
            results[path.name] = {
                "invoice": None,
                "flags": ["extraction failed"],
                "error": str(exc),
            }
            print(f"{path.name}: NEEDS REVIEW")
            print(f"  - extraction failed: {exc}")

    Path("results.json").write_text(
        json.dumps(results, indent=2, ensure_ascii=False),
        encoding="utf-8",
    )

    print("\nSaved results to results.json.")


if __name__ == "__main__":
    main()
```

## Validation behavior

The script checks:

- Whether line-item amounts add up to the subtotal
- Whether subtotal plus tax equals the total
- Whether extraction or validation failed

If an arithmetic check fails, it retries once with higher reasoning effort. A successful retry still requires human approval.

The checks do not verify vendor identity, tax rules, duplicate invoices, currency conversions, fraud indicators, or accounting treatment.

## Important safety notes

- Review every extracted field against the original invoice.
- Do not automatically post or pay an invoice based on model output.
- Treat `OK` as “arithmetic checks passed,” not “invoice is verified.”
- Do not upload invoices that you are not authorized to share with a third-party API.
- Review CometAPI’s and its upstream providers’ data-handling policies before sending confidential documents.
- Add `results.json`, API keys, and private invoices to `.gitignore`.

Example `.gitignore`:

```gitignore
.env
results.json
invoices/
```

## Pricing

Check the [current GPT-6.1 Terra pricing](https://www.cometapi.com/models/openai/gpt-6-1-terra/?utm_source=github&utm_medium=distribution&utm_campaign=bom&utm_content=gpt-6-1-terra) before estimating costs.

Your total may depend on:

- Input and output token rates
- Reasoning-token billing
- Image-processing charges
- Retries
- Context caching
- Provider-specific limits

## Links

- [GPT-6.1 Terra model page and pricing](https://www.cometapi.com/models/openai/gpt-6-1-terra/?utm_source=github&utm_medium=distribution&utm_campaign=bom&utm_content=gpt-6-1-terra)
- [CometAPI API documentation](https://apidoc.cometapi.com)

## License

MIT

````
