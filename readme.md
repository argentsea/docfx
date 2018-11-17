# ArgentSea Documentation

For a description of ArgentSea, please visit the the web page at http://www.argentsea.com.

This repostitory contains the markdown and links that enables DocFx to create the documentation site for ArgentSea. Documentation generation requires local access to other ArgentSea repostitories. These should each be sibling folders, with the names as:

* https://github.com/argentsea/docfx cloned to \ArgentSea.DocFx
* https://github.com/argentsea/shared cloned to \ArgentSea.Shared
* https://github.com/argentsea/sql cloned to \ArgentSea.Sql
* https://github.com/argentsea/pg cloned to \ArgentSea.Pg
* https://github.com/argentsea/pg cloned to \ArgentSea.Pg
* https://github.com/argentsea/docs cloned to \ArgentSea.DocFx\_site

DocFx should generate a web site to the “_site” folder. The _site folder is it’s own repostoriry; pushing that site to GitHub will publish the updated documentation.

This repository uses the MIT license.