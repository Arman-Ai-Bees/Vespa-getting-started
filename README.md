# Basic Text Search Application

## Install 
pip3 install pyvespa>=0.2.0 learntorank


## Define the application
`ApplicationPackage`
- Set of files in a specific strucutre
- Defines a deployable app
- It is a directory
- Contains:
    - Configurations
    - Components
    - Machine-learned models
- Consists of:
    - services.xml

### Schema  
Defines:  
- **Document Type**: Specifies the type of data stored.  
- **Rank Profiles**: Determines how documents are ranked.  
- **Document Summaries**: Structures for data retrieval.  

Details:  
- Schemas are stored as `.sd` files in the `schemas/` directory of the application package.  
- Inheritance is supported for document types, rank profiles, and document summaries.  

---

### Document  
- **Definition**: A document is the unit evaluated by the rank-profile and is returned in query results.  
- **Fields**: Documents have fields, which can be updated as full documents or individual fields.  
- **Relationships**: Documents can have relations, and field values can be imported from parent documents.  
- **Document ID**: Not included as a default field; must be explicitly added if needed.  

---

### Field  
- **Definition**: A field has a type (e.g., string, integer).  
- **Usage**: Fields can be written to, read from, and queried. Some fields can be synthetic, generated outside the document.  
- **Multivalue Fields**: Fields can hold single or multiple values (e.g., arrays, weighted sets, tensors). These can be used in grouping but cannot access attributes in maps/arrays of struct in ranking.  

#### Ranking with Multivalue Fields  
- Use `attribute(name).count` to rank based on the number of elements.  
- Filtering by element count is more expensive; consider creating a separate field for this purpose.  

#### Field Size  
- No max size for fields in bytes; larger fields (e.g., string, raw) can impact performance.  
- Use **summary classes** to limit fields returned in query responses.  
- Text fields are capped at `max-length` characters during indexing. Increase this to index large fields.  

---

### Struct  
- **Definition**: Groups one or more fields into a composite type that can be accessed like a single field.  
- **Example**:  

    ```text
    struct email {
        field sender type string {}
        field recipient type string {}
        field subject type string {}
        field content type string {}
    }
    field emails type array<email> {}
    ```
- **Usage**: Structs can be part of arrays or maps.  

---

### Struct-Field  
- **Definition**: Defines how specific fields in a struct should be indexed and searched.  
- **Example**:  

    ```text
    field emails type array<email> {
        indexing: summary
        struct-field content {
            indexing: attribute
            attribute: fast-search
        }
    }
    ```

---

### Indexing  
- **Purpose**: Configures data processing for a field during indexing.  
- **Key Instructions**:  
  - **index**: For unstructured text; creates a text index.  
  - **attribute**: For structured data; keeps fields in memory for grouping, sorting, and ranking.  
  - **summary**: Includes fields in document summaries for query results.  
- **Pipeline Semantics**: Supports chaining transformations, e.g., `summary | attribute | index`.  

#### Important Notes  
- Fields with both `attribute` and `index` use index mode for queries.  

---

### Match  
- Configures query item matching to fields (e.g., exact, prefix).  
- Use **sameElement()** to restrict matches to the same struct element in arrays/maps.  

---

### Fieldset  
- **Definition**: Groups fields for querying as a single unit.  
- **Example**:  

    ```text
    fieldset myset {
        fields: artist, title, album
    }
    ```

    Query example:  
    ```text
    vespa query "select * from sources * where myset contains 'bob'"
    ```

---

### Rank Profile  
- **Definition**: Specifies computations performed over documents during query matching.  
- **Purpose**: Optimizes document ranking based on defined criteria.  

---
```example
schema music {  
    # Schema: `music`
    # This schema defines a document type and specifies how fields, fieldsets, and ranking profiles are structured.

    document music {  
        # Document: `music`
        # Represents the unit of data storage and querying. Each document contains the following fields:

        field artist type string {  
            # Field: `artist`  
            # Represents the name of the artist.  
            # - **Type**: `string` (textual data)  
            # - **Indexing**:  
            #   - `summary`: Included in query results.  
            #   - `index`: Available for full-text search and ranking.  
            indexing: summary | index  
        }

        field artistId type string {  
            # Field: `artistId`  
            # A unique identifier for each artist.  
            # - **Type**: `string` (textual data)  
            # - **Indexing**:  
            #   - `summary`: Included in query results.  
            #   - `attribute`: Stored in memory for filtering and sorting.  
            # - **Match Mode**: `word` (matches individual words).  
            # - **Rank**: `filter` (used for filtering in ranking).  
            indexing: summary | attribute  
            match: word  
            rank: filter  
        }

        field title type string {  
            # Field: `title`  
            # Represents the title of the song.  
            # - **Type**: `string` (textual data)  
            # - **Indexing**:  
            #   - `summary`: Included in query results.  
            #   - `index`: Available for full-text search and ranking.  
            indexing: summary | index  
        }

        field album type string {  
            # Field: `album`  
            # Represents the name of the album the song belongs to.  
            # - **Type**: `string` (textual data)  
            # - **Indexing**:  
            #   - `index`: Available for full-text search only.  
            indexing: index  
        }

        field duration type int {  
            # Field: `duration`  
            # Represents the duration of the song (in seconds).  
            # - **Type**: `int` (numerical data)  
            # - **Indexing**:  
            #   - `summary`: Included in query results.  
            indexing: summary  
        }

        field year type int {  
            # Field: `year`  
            # Represents the release year of the song.  
            # - **Type**: `int` (numerical data)  
            # - **Indexing**:  
            #   - `summary`: Included in query results.  
            #   - `attribute`: Stored in memory for filtering and sorting.  
            indexing: summary | attribute  
        }

        field popularity type int {  
            # Field: `popularity`  
            # Represents the popularity score of the song.  
            # - **Type**: `int` (numerical data)  
            # - **Indexing**:  
            #   - `summary`: Included in query results.  
            #   - `attribute`: Stored in memory for filtering, ranking, and sorting.  
            indexing: summary | attribute  
        }
    }

    fieldset default {  
        # Fieldset: `default`  
        # Groups related fields for querying as a single unit.  
        # - **Fields**: `artist`, `title`, `album`  
        fields: artist, title, album  
    }

    rank-profile song inherits default {  
        # Rank Profile: `song`  
        # Defines the ranking logic for documents in this schema.  
        # Inherits the default ranking settings.

        first-phase {  
            # First-Phase Ranking:  
            # - Uses native ranking on `artist` and `title`.  
            # - Adds the `popularity` score if it is not NaN; otherwise, assigns a score of 0.  
            expression {  
                nativeRank(artist,title) +  
                if(isNan(attribute(popularity)) == 1, 0, attribute(popularity))  
            }  
        }  
    }
}
```
