---
layout: page
path: /postgraphile/smart-tags-file/
title: The postgraphile.tags.json5 file
---

When running PostGraphile in CLI mode, PostGraphile will automatically look for a `postgraphile.tags.json5` file in the current directory, and will process the tags and descriptions therein.

In library mode, you must add a plugin (see below) to source this file.

### Merging/Overriding

If you provide a description for an entity in `postgraphile.tags.json5` then that description will override any previous descriptions.

If you provide tags for an entity in `postgraphile.tags.json5`, those tags will be _merged_ with previous tags overriding tags with the same names but retaining other tags.

### File format

The file is in JSON5 (you can just use regular JSON if you prefer, but the extension must be `.json5`) and is formatted like this:

```json5
{
  version: 1,
  config: {
    /*
     * There can be entries here for:
     *
     * - `class`: for tables, composite types, views and materialized views
     * - `attribute`: for columns/attributes (of any 'class' type)
     * - `constraint`: for table constraints
     * - `procedure`: for functions/procedures
     */
    class: {
      /*
       * The next level describes the named type. We've just used the table
       * name `"post"` but it could be `"my_schema.post"` if you have multiple
       * tables with the same name and you don't want this rule to apply to
       * all of them.
       */
      post: {
        /*
         * This will override the description sourced from the PostgreSQL COMMENT.
         */
        description: "A post within our forum.",

        /*
         * Add tags specific to the 'post' table here. You can omit this if you
         * don't want to add any tags.
         */
        tags: {},

        /*
         * We've added a shortcut to class-types so you can tag/describe
         * columns at the same time of the class.
         */
        columns: {
          /*
           * Assuming `body` is one of the columns in the 'post' table.
           */
          body: {
            /*
             * Optional description, if provided overrides the PostgreSQL
             * `COMMENT ON COLUMN post.body`.
             */
            description: "The body of the post",
            tags: {
              /*
               * Here we indicate that the 'body' field will not be available
               * in the update mutation.
               */
              omit: "update",
            },
          },
        },
      },
    },
  },
}
```

### Library usage

PostGraphile library mode doesn't automatically import the `postgraphile.tags.json5` file for you, so you need to do a little more work.

For an example of a plugin that supports `postgraphile.tags.json5` and automatically detects changes to the file in **watch mode**, see [cli-tags.ts](https://github.com/graphile/postgraphile/blob/master/src/postgraphile/cli-tags.ts) in PostGraphile.

A basic smart tags plugin that doesn't require importing a JSON file might
look something like this:

```js
const { makeJSONPgSmartTagsPlugin } = require("graphile-utils");

module.exports = makeJSONPgSmartTagsPlugin({
  version: 1,
  config: {
    class: {
      post: {
        tags: {
          omit: "update",
        },
      },
    },
  },
});
```

A basic plugin that imports `postgraphile.tags.json5` might look something like this:

```js
const { readFileSync } = require("fs");
const { parse } = require("json5");
const { makeJSONPgSmartTagsPlugin } = require("graphile-utils");

const tagsJSON = parse(readFileSync("./postgraphile.tags.json5", "utf8"));
module.exports = makeJSONPgSmartTagsPlugin(tagsJSON);
```

You can load any of these plugins with the `appendPlugins` library option:

```js
const MySmartTagsPlugin = require("./MySmartTagsPlugin");
app.use(
  postgraphile(DATABASE_URL, SCHEMAS, {
    // ...
    appendPlugins: [MySmartTagsPlugin],
  })
);
```