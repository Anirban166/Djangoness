A collection of personal notes extracted from some of my work for NAU-CCL. Primarily focused on full-stack development using Django on the backend and a handful of other technologies (PostgreSQL, React, Bootstrap, AWS, Apache, etc.).

A few things that I found myself doing include:

- Generating rule-based images (QR codes for instance) on the fly and after a base64 decoding, serving them to the client's browser without associating them to a model (or without storing them in a database).
- Using a defaultdict (from collections) for returning context from a view that is to be modified externally (i.e., certain entries in the dictionary are not explicitly assigned within the view code itself).
- Switching from Apple Silicon (arm64 architecture) to Intel's instruction set (Rosetta handles the translations), for e.g. the time I used PyQt5 binaries for running a native application with a Qt-based GUI. (easiest way being `arch -x86_64 zsh` for a terminal and nope, duplicating finders isn't a feature in OS X anymore)
- Automating the setup of outbound rules for different ports used in production (separate specifications for the Django and Apache servers on an AWS EC2 for instance).
- Chaining shell commands with python code (while following best practices for spawned processes such as setting `shell=false`) to achieve tasks such as finding and terminating a specific actively running python process that is associated with an endpoint:
```py
if endpoint.pid is not None:
  ...
  if "python" in psutil.Process(endpoint.pid).name().lower():
    processCommand = subprocess.run(['ps', '-p', str(endpoint.pid), '-o', 'args'], stdout=subprocess.PIPE).stdout.decode('utf-8').rstrip()
    if processCommand[-filenameLength:] == "<filename>.py":
      os.system('kill -9 {0}'.format(endpoint.pid))
# The above runs ps -p <PID> -o args as a subprocess in shell, gets the result piped back into python, removes the encoding, strips the trailing whitespace, and returns a string where the suffix includes the desired script's name.
```
- Using a live server (VSC extension) to get prompt visual feedback from CSS-based changes for templates, as static files aren't reloaded every time (clearing the cache time and again isn't very convenient).
- Sending a `SIGKILL` to the port that runs my Django server in the case I unwittingly suspend the process instead of interrupting it: (i.e., a <kbd>Ctrl</kbd><kbd>+</kbd><kbd>Z</kbd> instead of a <kbd>Ctrl</kbd><kbd>+</kbd><kbd>C</kbd>)
```sh
sudo lsof -t -i tcp:$portNumber | xargs kill -9
```
- Tailing human-readable files and streaming their contents in a browser via a websocket.
- Implementing sorting, search, and pagination for HTML tables (made easy thanks to the jQuery plugin DataTables).
- Using JavaScript or Python over SQL to avoid making queries to the database for such operations. For instance, here is how I used Python's list-based sort method to order the entries for a `@property` (decorator pattern) attribute of my model which is dependent on the existence of a field:
```py
list(Model.objects.exclude(<field>__isnull=True)).sort(key=operator.attrgetter('property'), reverse=True)
```
This is as opposed to using `order_by()` which is only applicable for fields in a model, and is SQL-based (slower). Later, I refactored all of the sorting to be done using JavaScript instead since sending a POST request would mean page refreshes (which can invalidate other JavaScript operations) with back-and-forth communication between my backend (Django server) and the localhost port I’m using (client in production). This makes things much faster, apart from clearing that drawback entirely. For searching fields as well, I resorted to using JS and discarded the former approaches that I tried successfully (`Q`, `django-filters`, and `SearchVector`).
- Automating dependent empty fields to be filled and modified by user input to other fields. A note here is that overriding the save method for models should be avoided unless there is a good reason to, and even then it should be accompanied by constraints since duplicate entries are created every time objects are saved (easiest way to check being the creation of such instances via the administration panel).
- Using Postgres.app instead of the brew bottle for PostgreSQL along with related libraries as the Django server requires the host to accept TCP/IP connections to the database, and even after carefully configuring my homebrew-based setup (correcting symlinks and permissions for instance), I experienced recurring issues with port connections and existing postmaster process IDs at times with the latter form of psql installation. (documented all the solutions to these pain points and the reasons for the switch, so that developers can be wary of such database-related quirks when using OS X, apart from the slightly more cumbersome setup of resolving virtual hosts with the `httpd` bottle in place of the pre-installed and conflicting Apache system)
- Abiding by a concrete workflow for adding fields to my models and setting attributes to them, as I wouldn't want to end up deleting the migration files in case of an unresolvable database-related issue in production. In the extreme case where it has to be done locally, my workflow would be the ordinary 'delete the files in `<appName>/migrations/` excluding `__init__.py`', with a follow-up of dropping the concerned table(s) and deleting the application before rerunning the migration: 
```sh
drop table <tableName>;
delete from django_migrations where app='<appname>';
```
Since Django's migrations do not auto-magically rebuild schemas, performing a drop on a table instead of a delete can lead to a potential issue of the deleted table not getting created upon running `makemigrations` and `migrate` thereafter (with `--fake <app> zero` as well), even though the SQL statements have mention of it to be created. Best to be wary of this and go with the appropriate workflow of keeping the table intact and deleting the entries only, if a reset on the data is to be achieved. Running `migrate` can also apply changes for files that have been there but not been applied previously since Django doesn't know to replace migration files (looking at the date of modification for those files would help).
- Incorporating NoSQL data from Firebase (both from Firestore via `firebase_admin` and a real-time database via `pyrebase`), with non-standard formats such as GeoJSON.
- Basic Django stuff such as:
  - Making use of Django's REST API to check the JSON that the serializers return (to be fetched and used at the frontend, like for e.g. in a React component).
  - Strictly abiding by the use of the `{% %}` token to pass in view names and corresponding variables within when using forms, as Django would otherwise construct relative URLs when simply using the view name as the action, leading to weird path constructions at times.
  - Implementing authentication and modals with forms.
  - Using template inheritance (to avoid code repetition) and a custom user model.
- Optimizing image load times by having thumbnails of them show up in place, only loading the original/full-resolution images when specifically previewed for an individual one. In essence, this meant that the downscaling operation that had to be performed on the backend has to be fast enough to scale. Thus, after a small bit of research and trial-and-error with image processing libraries, I resorted to using a SIMD version of the `Pillow` library to rescale images, and as expected, it works faster than the default PIL or other alternatives that I've tried (`scikit-image` for instance). Here's the code snippet I initially came up with for the same:
```sh
import glob, os
# pip install pillow-simd
from PIL import Image

def generateThumbnails():
    thumbnailSize = 256, 256
    thumbnailSuffix = "thumb.png"
    for sites in ([x[0] for x in os.walk("../../static/images/ifson/")])[1:]:
        JPGFiles = sites + "/*.jpg"
        PNGFiles = sites + "/*.png"
        for infile in [f for f_ in [glob.glob(e) for e in (JPGFiles, PNGFiles)] for f in f_]:
            if not infile.endswith(thumbnailSuffix):
                file, ext = os.path.splitext(infile)
                with Image.open(infile) as img:
                    img.thumbnail(thumbnailSize)
                    img.save(file + thumbnailSuffix)
```
Note that `glob` does not have a way to match filenames via the use of regular expressions (thus, I couldn't use negative lookaheads, which was my first goto for simplifying the filtering).
For instance, `"(?<!\/thumb).(png|jpg)$"` or a simple exclusion like `"(?:thumb)"` won't work. (Also note that I'm excluding the first directory for my `os.walk` since that's the root)

Post this change, I updated the serializer that dealt with image-specific fields to have a separate URL-field for the thumbnail, and ensured that the frontend (React) used those as the source for display in general (for faster load times again), otherwise, the original ones if an image is previewed specifically.
Note that if the image data that I was working with was not sensitive, cloud-based services such as ImageKit (where one can just append resize parameters to the URLs) were an option. But since a core part of my project deliverable included the images I dealt with, I discarded such alternatives (since that would mean having to upload images to their domain and add a third-party dependency) and did it manually.

PS: I'm on a hiatus from writing blog posts. For the moment, I'm thinking of instead having standalone open-source repositories such as this for sharing some of my recent experiences in building software or playing with technologies.