# Database

Este repositorio contiene un script que recorre la carpeta de Google Drive solicitada y genera una base de datos SQLite con los títulos de todos los documentos encontrados.

## Requisitos previos

* Python 3.10+
* `pip`

## Instalación de dependencias

```bash
pip install -r requirements.txt
```

## Uso

Ejecuta el script `scripts/build_database.py` para recorrer la carpeta y crear la base de datos:

```bash
python scripts/build_database.py
```

Por defecto, se analiza la carpeta pública indicada en el script y la base de datos se genera en `data/documents.db`. Puedes modificar la URL de la carpeta con el argumento `--url` y la ruta de salida con `--database`.

La tabla `documents` contiene las columnas:

* `id`: identificador autoincremental.
* `title`: título del documento (nombre tal como aparece en Google Drive).
* `relative_path`: ruta del archivo relativa al directorio raíz analizado.
* `file_id`: identificador del archivo en Google Drive.
* `url`: enlace directo al elemento en Google Drive.

## Dependencias

Consulta `requirements.txt` para ver las bibliotecas utilizadas.
data/documents.db
Nuevo
No se muestra el archivo binario
requirements.txt
Nuevo
+2
-0

beautifulsoup4>=4.12.3
requests>=2.31.0
scripts/build_database.py
Nuevo
+144
-0

#!/usr/bin/env python3
"""Crea una base de datos SQLite con los títulos de un folder de Google Drive."""

from __future__ import annotations

import argparse
import sqlite3
from dataclasses import dataclass
from pathlib import Path
from typing import Generator, Iterable, List, Set
from urllib.parse import parse_qs, urlparse

import requests
from bs4 import BeautifulSoup


EMBED_URL = "https://drive.google.com/embeddedfolderview?id={folder_id}"


@dataclass
class Document:
    title: str
    relative_path: str
    file_id: str
    url: str


def extract_folder_id(url: str) -> str:
    parsed = urlparse(url)
    if parsed.query:
        params = parse_qs(parsed.query)
        if "id" in params:
            return params["id"][0]
    parts = [p for p in parsed.path.split("/") if p]
    if "folders" in parts:
        return parts[parts.index("folders") + 1]
    return url.strip()


def parse_entry_type(href: str) -> tuple[str, str | None]:
    parsed = urlparse(href)
    parts = [p for p in parsed.path.split("/") if p]
    if "folders" in parts:
        return "folder", parts[parts.index("folders") + 1]
    if "file" in parts and "d" in parts:
        return "file", parts[parts.index("d") + 1]
    for marker in ("document", "spreadsheets", "presentation", "forms", "drawings"):
        if marker in parts and "d" in parts:
            return "file", parts[parts.index("d") + 1]
    if parsed.query:
        params = parse_qs(parsed.query)
        if "id" in params:
            return "file", params["id"][0]
    return "unknown", None


def fetch_folder(
    session: requests.Session,
    folder_id: str,
    parents: List[str],
    visited: Set[str],
) -> Generator[Document, None, None]:
    if folder_id in visited:
        return
    visited.add(folder_id)
    url = EMBED_URL.format(folder_id=folder_id)
    response = session.get(url, timeout=60)
    response.raise_for_status()
    soup = BeautifulSoup(response.text, "html.parser")
    entries = soup.select(".flip-entry")
    for entry in entries:
        title = entry.select_one(".flip-entry-title").get_text(strip=True)
        href = entry.select_one("a")["href"]
        entry_type, entry_id = parse_entry_type(href)
        if entry_type == "folder" and entry_id:
            yield from fetch_folder(session, entry_id, parents + [title], visited)
        elif entry_type == "file" and entry_id:
            relative_path = "/".join(parents + [title])
            yield Document(title=title, relative_path=relative_path, file_id=entry_id, url=href)


def crawl_documents(folder_url: str) -> Iterable[Document]:
    session = requests.Session()
    root_id = extract_folder_id(folder_url)
    return fetch_folder(session, root_id, [], set())


def build_database(db_path: Path, documents: Iterable[Document]) -> int:
    db_path.parent.mkdir(parents=True, exist_ok=True)
    with sqlite3.connect(db_path) as conn:
        cursor = conn.cursor()
        cursor.execute(
            """
            CREATE TABLE IF NOT EXISTS documents (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                title TEXT NOT NULL,
                relative_path TEXT NOT NULL,
                file_id TEXT NOT NULL,
                url TEXT NOT NULL
            )
            """
        )
        cursor.execute("DELETE FROM documents;")
        rows = []
        count = 0
        for doc in documents:
            rows.append((doc.title, doc.relative_path, doc.file_id, doc.url))
            count += 1
            if count % 200 == 0:
                print(f"Procesados {count} documentos...")
        cursor.executemany(
            "INSERT INTO documents (title, relative_path, file_id, url) VALUES (?, ?, ?, ?)",
            rows,
        )
        conn.commit()
    return count


def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument(
        "--database",
        type=Path,
        default=Path("data/documents.db"),
        help="Ruta al archivo SQLite que se generará.",
    )
    parser.add_argument(
        "--url",
        type=str,
        default="https://drive.google.com/drive/folders/1gk0l31EilFZEPYhYjaMwsKtrIjJK6qCO",
        help="URL o identificador del folder de Google Drive a procesar.",
    )
    return parser.parse_args()


def main() -> None:
    args = parse_args()
    documents = crawl_documents(args.url)
    total = build_database(args.database, documents)
    print(f"Base de datos creada en {args.database} con {total} documentos.")


if __name__ == "__main__":
    main()
