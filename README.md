# Webserv

A C++98 HTTP/1.1 server that serves static assets, runs CGI programs, and exposes a configurable routing layer via an Nginx-style configuration syntax. The project ships with a sample website, comprehensive integration tests, and helper scripts for uploading files, deleting resources, and exercising CGI scripts.

---

## Demo Preview

Here is a view of the actual website served by **webserv**:

<p align="center">
  <img src="https://raw.githubusercontent.com/emily-cloud/HTTP-webserve/main/images/Screenshot%202025-12-31%20at%2011.27.31.png" alt="Home Page" width="75%">
</p>

---

## Quick Start

1. **Build**
   ```bash
   make
   ```
   The default target compiles `webserv` and immediately runs the pytest suite (see `Makefile`).

2. **Run with the default configuration**
   ```bash
   ./webserv config/default.conf
   ```
   Omit the argument to fall back to `Constants::default_config_file` (currently `config/default.conf`).

3. **Helper targets**
   - `make run` – build then start the server with `config/default.conf`
   - `make valrun` – run the server under Valgrind with strict leak checks
   - `make clean | make fclean | make re` – housekeeping targets

---

## Features

- **Multi-server, multi-port**: Each `server {}` block can listen on one or more ports; the default config exposes five ports that serve distinct document roots.
- **Static file hosting**: Configurable `root`, `index`, `autoindex`, and MIME type detection for HTML, CSS, JS, images, and arbitrary assets under `html/`.
- **Upload & DELETE workflow**: Toggle `file_upload on` per `location` to accept raw uploads, support URL-encoded names (including spaces), and delete with HTTP `DELETE` ([tests/integration/test_uploads.py](tests/integration/test_uploads.py)).
- **CGI gateway**: Run Python or Perl handlers located in `html/www1/cgi-bin/`, passing the full CGI/1.1 environment, multipart bodies, chunked transfer data, and guarding against disallowed extensions ([tests/integration/test_cgi.py](tests/integration/test_cgi.py), [tests/integration/test_bad_cgi_file_names_and_extensions.py](tests/integration/test_bad_cgi_file_names_and_extensions.py)).
- **Custom error pages & redirects**: Map status codes to HTML templates and define `return 3xx` directives per location ([config/default.conf](config/default.conf)).
- **Cookie & JSON helpers**: Sample endpoint `/api/update-cookie/...` demonstrates authoring multiple `Set-Cookie` headers and JSON responses ([tests/integration/test_cookies.py](tests/integration/test_cookies.py)).
- **Robust configuration parsing**: Invalid or empty configs fail fast, as shown in [tests/integration/test_config_errors.py](tests/integration/test_config_errors.py).

---

## Static Assets, Uploads, and CGI

Static HTML lives under `html/www1/`, `html/www2/`, and `html/www3/`, with dedicated assets for cookies, documentation, denial pages, and sample uploads. Uploads land in `html/www1/upload/` by default; `file_upload on` is required for both POST uploads and DELETE clean-up.

### Upload Demo

Upload page:

<p align="center">
  <img src="https://raw.githubusercontent.com/emily-cloud/HTTP-webserve/main/images/Screenshot%202025-12-31%20at%2011.27.41.png" alt="Upload Page" width="75%">
</p>

After a file has been uploaded:

<p align="center">
  <img src="https://raw.githubusercontent.com/emily-cloud/HTTP-webserve/main/images/Screenshot%202025-12-31%20at%2011.28.03.png" alt="Upload Result" width="75%">
</p>

### CGI Demo

The CGI scripts reside in `html/www1/cgi-bin/`. The server injects the CGI/1.1 variables (`PATH_INFO`, `QUERY_STRING`, `CONTENT_LENGTH`, etc.) and enforces execution timeouts (`Constants::cgi_child_timeout`).

Example CGI response:

<p align="center">
  <img src="https://raw.githubusercontent.com/emily-cloud/HTTP-webserve/main/images/Screenshot%202025-12-31%20at%2011.28.13.png" alt="CGI Output" width="75%">
</p>

Example invocations:

```bash
curl http://localhost:4244/cgi/hello.py
curl -X POST -F "text_field=demo" -F "file_upload=@tests/report.txt" http://localhost:4244/cgi/hello.py
curl http://localhost:4244/cgi/hello.pl
```

---

## Custom Error Pages & Redirects

Map status codes to HTML templates and define `return 3xx` directives per location.

Redirect example:

<p align="center">
  <img src="https://raw.githubusercontent.com/emily-cloud/HTTP-webserve/main/images/Screenshot%202025-12-31%20at%2011.28.27.png" alt="Redirect Demo" width="75%">
</p>

Coffee / tea demo:

<p align="center">
  <img src="https://raw.githubusercontent.com/emily-cloud/HTTP-webserve/main/images/Screenshot%202025-12-31%20at%2011.28.36.png" alt="Coffee Demo" width="75%">
</p>

Custom `418 I'm a teapot` page:

<p align="center">
  <img src="https://raw.githubusercontent.com/emily-cloud/HTTP-webserve/main/images/Screenshot%202025-12-31%20at%2011.28.41.png" alt="Teapot Error" width="75%">
</p>

---

## Cookie & JSON Helpers

Sample endpoint `/api/update-cookie/...` demonstrates authoring multiple `Set-Cookie` headers and JSON responses, and the sample cookie demo page shows a simple interactive counter.

Cookie demo:

<p align="center">
  <img src="https://raw.githubusercontent.com/emily-cloud/HTTP-webserve/main/images/Screenshot%202025-12-31%20at%2011.28.50.png" alt="Cookie Demo" width="75%">
</p>

---

## Project Layout

```text
HTTP-webserve/
├── Makefile                # Build + test automation
├── src/                    # C++ sources (HTTPServer, Parser, CGI, etc.)
├── include/                # Shared headers
├── config/default.conf     # Example multi-server configuration
├── html/                   # Static site, error pages, CGI scripts, uploads
└── tests/                  # Pytest integration suite + config fixtures
```

Key entry points:
- `src/main.cpp` – parses CLI args, loads config, and starts `HTTPServer`.
- `src/HTTPServer.*` – accepts sockets, dispatches requests, and coordinates responses.
- `src/Config.*` / `src/Parser*.cpp` – tokenize and validate the custom config syntax.
- `html/www1/cgi-bin/` – Python/Perl CGI samples such as `hello.py`, `hello.pl`, `fileupload.py`.

---

## Configuration Primer

The configuration format mirrors a subset of Nginx. Blocks are delimited by braces, statements end with semicolons, and indentation is optional. The default file illustrates the available directives:

```conf
http {
    maxBodySize 100000000; mandatory

    error_pages {
        400 html/www1/error_pages/400.html
        403 html/www1/error_pages/403.html
        ...
    }

    server {
        listen 4254;
        server_name myWebserver someWebserver;
        root html/www1/;

        location /here {
            acceptedMethods GET POST DELETE;
        }

        location /upload {
            file_upload on;
        }

        cgi {
            cgi_path_alias /cgi "/cgi-bin";
            upload_dir html/www1/upload;
            file_extension .cgi .py;
            acceptedMethods GET POST DELETE;
        }
    }

    server {
        listen 4256;
        root ./htmltest/www2/;
    }
}
```

### Directive Highlights
- `listen <port>;` – Bind sockets (repeat to reuse the same block for multiple ports).
- `server_name` – Apply named-virtual-host routing.
- `root` / `index` – Define document roots and default documents.
- `location <path> { ... }` – Override behavior per prefix; supports `acceptedMethods`, `autoindex`, `file_upload`, `return`, and nested `cgi` configs.
- `cgi { ... }` – Attach CGI interpreters with path aliases, upload directories, and allowed extensions.
- `error_pages { code path }` – Map status codes to HTML templates.

Copy `config/default.conf`, trim the unused servers, and adapt roots and ports to your environment. If a directive is marked `mandatory`, the parser will reject the file when it is missing.

---

## Upload & Delete Cheatsheet

```bash
# Upload a text file
curl -X POST --data-binary @tests/uploadtest.txt \
     -H "Content-Type: text/plain" \
     http://localhost:4244/upload/test.txt

# Upload with URL-encoded name (spaces)
curl -X POST --data-binary @tests/screenshot\ +\ space.png \
     -H "Content-Type: image/png" \
     http://localhost:4244/upload/screenshot%20%2B%20space.png

# Delete the resource
curl -X DELETE http://localhost:4244/upload/test.txt
```

---

## Testing

Integration tests live in `tests/integration/` and spin up the compiled binary with different configs:

```bash
make test          # builds, ensures venv, runs pytest
./venv/bin/pytest tests/integration/test_cgi.py -k post
```

Fixtures in `tests/integration/conftest.py` manage server lifecycles for:
- Normal config (`tests/config/default.conf`)
- Error page config (`tests/config/error_pages.conf`)
- Redirection config (`tests/config/redirections.conf`)
- Invalid/empty configs (expect startup failure)

---

## Troubleshooting

- **Port already in use**: Stop existing server instances or adjust the `listen` directives.
- **Config fails to load**: The parser is strict about semicolons, block braces, and mandatory directives; check stderr for precise hints.
- **404s on uploads**: Ensure the target `location` enables `file_upload on` and that you are writing inside the configured `root`.
- **CGI timeout (504)**: Long-running scripts (see `cgi/endlessloop.py`) are terminated after `Constants::cgi_child_timeout` seconds.

---

## License

This project is licensed for educational use by the following contributors:

- Huayun AI  
- Laurent Brusa  
- Rufus Lane 
- 42 Berlin Webserv Team  

Redistribution, commercial use, or modification outside the 42 school environment is prohibited without explicit permission from all contributors.

