## Sending Http Requests

```
package main

import (
    "fmt",
    "net/http"
)

func main() {
    resp, err := http.Get("<api_url>")
    if err != nil {
        fmt.Println("Error: ", err)
        return
    }

    if resp.StatusCode != http.StatusOK {
        fmt.Printf("Error: bad status - %s\n", resp.Status)
        return
    }

    ctype := resp.Header.Get("Content-Type")
    fmt.Println("Content-Type: ", ctype)

    io.Copy(os.Stdout, resp.Body)
}
```

> [!NOTE]
> In `resp.Header.Get("<header-name>")`, `<header-name>` is case-insensitive and if we have multiple headers with same name, it will extract them all.

## JSON APIs

- API -
    - `Marshal` - JSON -> []byte -> Go
    - `Unmarshal` - Go -> []byte -> JSON
    - `Decoder` - JSON -> io.Reader -> Go
    - `Encoder` - Go -> io.Writer -> JSON

- Parsing JSON in above example -
```
var reply struct {
    Name string
    Age int
    BirthYear int `json:"year_of_birth"`
}

decoder := json.NewDecoder(resp.Body)

if err := decoder.Decode(&reply); err != nil {
    fmt.Println("Error: ", err)
    return
}

fmt.Println(reply.Name, reply.Age)
```

- `Name` and `Age` fields, as defined in `struct` will be extracted from the JSON and are case-insensitive.

> [!NOTE]
> If a field is not present in JSON, it will NOT throw any errors, instead will just ignore it or return default value of the primitive type (for eg, 0 in case of Int)

- `json:"year_of_birth"` is called a "field tag" that tells Go to extract `year_of_birth` field from JSON and store it in `BirthYear`.

