# MDX Tricks

A collection of useful MDX tricks

## Top-Level Table of Contents from Imported MDX Headings (Next.js)

[`@jsdevtools/rehype-toc`](https://github.com/JS-DevTools/rehype-toc) is a rehype plugin for adding a table of contents to your Markdown and MDX documents, which works well for simple use cases, eg:

The following MDX...

```mdx
# Apple Pie Recipe

## Filling

### Preparing the apples

### Preparing the spice mix

## Crust

### Preparing the dough

### The criss-cross top
```

...will result in HTML with a table of contents:

```html
<nav class="toc">
  <ol class="toc-level toc-level-1">
    <li class="toc-item toc-item-h1">
      <a class="toc-link toc-link-h1" href="#apple-pie-recipe">
        Apple Pie Recipe
      </a>

      <ol class="toc-level toc-level-2">
        <li class="toc-item toc-item-h2">
          <a class="toc-link toc-link-h2" href="#filling">
            Filling
          </a>

          <ol class="toc-level toc-level-3">
            <li class="toc-item toc-item-h3">
              <a class="toc-link toc-link-h3" href="#preparing-the-apples">
                Preparing the apples
              </a>
            </li>
            <li class="toc-item toc-item-h3">
              <a class="toc-link toc-link-h3" href="#preparing-the-spice-mix">
                Preparing the spice mix
              </a>
            </li>
          </ol>
        </li>

        <li class="toc-item toc-item-h2">
          <a class="toc-link toc-link-h2" href="#crust">
            Crust
          </a>

          <ol class="toc-level toc-level-3">
            <li class="toc-item toc-item-h3">
              <a class="toc-link toc-link-h3" href="#preparing-the-dough">
                Preparing the dough
              </a>
            </li>
            <li class="toc-item toc-item-h3">
              <a class="toc-link toc-link-h3" href="#the-criss-cross-top">
                The criss-cross top
              </a>
            </li>
          </ol>
        </li>
      </ol>
    </li>
  </ol>
</nav>

<h1 id="apple-pie-recipe">Apple Pie Recipe</h1>

<h2 id="filling">Filling</h2>

<h3 id="preparing-the-apples">Preparing the apples</h3>

<h3 id="preparing-the-spice-mix">Preparing the spice mix</h3>

<h2 id="crust">Crust</h2>

<h3 id="preparing-the-dough">Preparing the dough</h3>

<h3 id="the-criss-cross-top">The criss-cross top</h3>
```

One place where `@jsdevtools/rehype-toc` (and the other existing remark and rehype plugins for tables of contents) fall down is imports of MDX content, eg:

If the "Filling" section (or the whole "Apple Pie Recipe" section) is moved out into its own file...

```mdx
import Filling from './filling.mdx';

# Apple Pie Recipe

<Filling />

## Crust

### Preparing the dough

### The criss-cross top
```

...[multiple tables of content get generated](https://github.com/orgs/mdx-js/discussions/2526) - see the "Filling" table of contents above the `<h2>`:

```html
<nav class="toc">
  <ol class="toc-level toc-level-1">
    <li class="toc-item toc-item-h1">
      <a class="toc-link toc-link-h1" href="#apple-pie-recipe">
        Apple Pie Recipe
      </a>

      <ol class="toc-level toc-level-2">
        <li class="toc-item toc-item-h2">
          <a class="toc-link toc-link-h2" href="#crust">
            Crust
          </a>

          <ol class="toc-level toc-level-3">
            <li class="toc-item toc-item-h3">
              <a class="toc-link toc-link-h3" href="#preparing-the-dough">
                Preparing the dough
              </a>
            </li>
            <li class="toc-item toc-item-h3">
              <a class="toc-link toc-link-h3" href="#the-criss-cross-top">
                The criss-cross top
              </a>
            </li>
          </ol>
        </li>
      </ol>
    </li>
  </ol>
</nav>

<h1 id="apple-pie-recipe">Apple Pie Recipe</h1>

<nav class="toc">
  <ol class="toc-level toc-level-2">
    <li class="toc-item toc-item-h2">
      <a class="toc-link toc-link-h2" href="#filling">
        Filling
      </a>

      <ol class="toc-level toc-level-3">
        <li class="toc-item toc-item-h3">
          <a class="toc-link toc-link-h3" href="#preparing-the-apples">
            Preparing the apples
          </a>
        </li>
        <li class="toc-item toc-item-h3">
          <a class="toc-link toc-link-h3" href="#preparing-the-spice-mix">
            Preparing the spice mix
          </a>
        </li>
      </ol>
    </li>
  </ol>
</nav>

<h2 id="filling">Filling</h2>

<h3 id="preparing-the-apples">Preparing the apples</h3>

<h3 id="preparing-the-spice-mix">Preparing the spice mix</h3>

<h2 id="crust">Crust</h2>

<h3 id="preparing-the-dough">Preparing the dough</h3>

<h3 id="the-criss-cross-top">The criss-cross top</h3>
```

To avoid multiple tables of contents on a single HTML page, a table of contents can be built dynamically using React Context. The following example of this approach uses Next.js (App Router with React Server Components) and Tailwind CSS.

First, set up the components and Context:

`TableOfContents.tsx`

```tsx
'use client';

import type { HTMLAttributes, ReactNode } from 'react';
import { createContext, useContext, useEffect, useState } from 'react';

function findParent(
  items: TableOfContentsItem[],
  newItem: TableOfContentsItem,
) {
  const lastItem = items.at(-1);

  if (!lastItem) {
    throw new Error(
      'No table of contents leaf item found with previous heading level',
    );
  }

  if (lastItem.headingLevel === newItem.headingLevel - 1) {
    return lastItem.items;
  }

  return findParent(lastItem.items, newItem);
}

type TableOfContentsContext = {
  tableOfContents: {
    ids: string[];
    items: TableOfContentsItem[];
  };
  addItem: (headingLevel: number, id?: string, children?: ReactNode) => void;
};

type TableOfContentsItem = {
  headingLevel: number;
  id: string;
  children: ReactNode;
  items: TableOfContentsItem[];
};

const tableOfContentsContext = createContext<TableOfContentsContext>({
  tableOfContents: { ids: [], items: [] },
  addItem: () => {},
});

const TableOfContentsContextProvider = tableOfContentsContext.Provider;

export function useTableOfContents(
  partialItem?: Pick<TableOfContentsItem, 'headingLevel' | 'children'> & {
    id?: string;
  },
) {
  const context = useContext(tableOfContentsContext);
  const addItem = context.addItem;

  useEffect(() => {
    if (partialItem) {
      addItem(partialItem.headingLevel, partialItem.id, partialItem.children);
    }
  }, [partialItem, addItem]);

  return context;
}

type TableOfContentsProviderProps = {
  children: ReactNode;
};

export function TableOfContentsProvider(props: TableOfContentsProviderProps) {
  const [tableOfContents, setTableOfContents] = useState<
    TableOfContentsContext['tableOfContents']
  >({
    ids: [],
    items: [],
  });

  function addItem(headingLevel: number, id?: string, children?: ReactNode) {
    if (typeof id !== 'string' || !id) {
      throw new Error('id is required');
    }

    setTableOfContents((prevTableOfContents) => {
      if (headingLevel !== 2 && prevTableOfContents.items.length === 0) {
        throw new Error(`First heading with id ${id} is not h2`);
      }

      if (prevTableOfContents.ids.includes(id)) return prevTableOfContents;

      const newTableOfContents = { ...prevTableOfContents };

      const newItem = {
        headingLevel,
        id,
        children,
        items: [],
      };

      newTableOfContents.ids.push(newItem.id);

      const parent =
        newItem.headingLevel === 2
          ? newTableOfContents.items
          : findParent(newTableOfContents.items, newItem);

      parent.push(newItem);

      return newTableOfContents;
    });
  }

  return (
    <TableOfContentsContextProvider value={{ tableOfContents, addItem }}>
      {props.children}
    </TableOfContentsContextProvider>
  );
}

type OlProps = {
  items: TableOfContentsItem[];
};

function Ol(props: OlProps) {
  return (
    <ol>
      {props.items.map((item) => (
        <li key={`item-${item.id}`}>
          {item.children}
          {item.items.length > 0 && <Ol items={item.items} />}
        </li>
      ))}
    </ol>
  );
}

export function TableOfContents() {
  const { tableOfContents } = useTableOfContents();

  return (
    <nav>
      <Ol items={tableOfContents.items} />
    </nav>
  );
}

export function H1ForTableOfContents({
  id,
  children,
  ...props
}: HTMLAttributes<HTMLHeadingElement>) {
  useTableOfContents({ headingLevel: 1, id, children });
  return (
    <h1 id={id} {...props}>
      {children}
    </h1>
  );
}

export function H2ForTableOfContents({
  id,
  children,
  ...props
}: HTMLAttributes<HTMLHeadingElement>) {
  useTableOfContents({ headingLevel: 2, id, children });
  return (
    <h2 id={id} {...props}>
      {children}
    </h2>
  );
}

export function H3ForTableOfContents({
  id,
  children,
  ...props
}: HTMLAttributes<HTMLHeadingElement>) {
  useTableOfContents({ headingLevel: 3, id, children });
  return (
    <h3 id={id} {...props}>
      {children}
    </h3>
  );
}

export function H4ForTableOfContents({
  id,
  children,
  ...props
}: HTMLAttributes<HTMLHeadingElement>) {
  useTableOfContents({ headingLevel: 4, id, children });
  return (
    <h4 id={id} {...props}>
      {children}
    </h4>
  );
}

export function H5ForTableOfContents({
  id,
  children,
  ...props
}: HTMLAttributes<HTMLHeadingElement>) {
  useTableOfContents({ headingLevel: 5, id, children });
  return (
    <h5 id={id} {...props}>
      {children}
    </h5>
  );
}

export function H6ForTableOfContents({
  id,
  children,
  ...props
}: HTMLAttributes<HTMLHeadingElement>) {
  useTableOfContents({ headingLevel: 6, id, children });
  return (
    <h6 id={id} {...props}>
      {children}
    </h6>
  );
}
```

Then, in your page (can be a React Server Component), import your desired MDX file and pass in the table of contents components in using the `components` prop:

```tsx
import type { MDXComponents } from 'mdx/types.js';
import type { JSX } from 'react';
import {
  H1ForTableOfContents,
  H2ForTableOfContents,
  H3ForTableOfContents,
  H4ForTableOfContents,
  H5ForTableOfContents,
  H6ForTableOfContents,
  TableOfContents,
  TableOfContentsProvider,
} from './TableOfContents.tsx';

type MdxModule = {
  default: (
    props: {
      readonly components?: MDXComponents | undefined;
      params: {
        slug: string;
      };
    },
  ) => JSX.Element;
};

type Props = {
  params: {
    slug: string;
  };
};

export default async function Page(props: Props) {
  let curriculumModule;

  try {
    curriculumModule = (await import(
      `./content/${props.params.slug}/index.mdx`
    )) as MdxModule
  } catch {
    notFound();
  }

  const MDXContent = curriculumModule.default;

  return (
    <TableOfContentsProvider>
      <details className="mt-[-0.6rem]">
        <summary>Table of Contents</summary>
        <TableOfContents />
      </details>

      <MDXContent
        params={props.params}
        components={{
          h1: H1ForTableOfContents,
          h2: H2ForTableOfContents,
          h3: H3ForTableOfContents,
          h4: H4ForTableOfContents,
          h5: H5ForTableOfContents,
          h6: H6ForTableOfContents,
        }}
      />
    </TableOfContentsProvider>
  );
}
```
