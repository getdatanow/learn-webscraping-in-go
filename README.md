# Learning Golang Through Web Scraping

Web scraping is a practical way to learn programming, as it requires working with several fundamental concepts like HTTP requests, data parsing, and structured output. In this tutorial, we'll use web scraping as a vehicle to learn Go programming, focusing on two popular libraries: Colly and goquery.

## Table of Contents
1. [Introduction to Golang](#introduction-to-golang)
2. [Setting Up Your Environment](#setting-up-your-environment)
3. [Web Scraping Basics](#web-scraping-basics)
4. [Getting Started with goquery](#getting-started-with-goquery)
5. [Building a Scraper with Colly](#building-a-scraper-with-colly)
6. [Advanced Techniques](#advanced-techniques)
7. [Project: Building a Complete Web Scraper](#project-building-a-complete-web-scraper)
8. [Best Practices and Considerations](#best-practices-and-considerations)

## Introduction to Golang

Go (or Golang) is a statically typed, compiled language designed by Google. It's known for its simplicity, efficiency, and built-in support for concurrency. These features make it an excellent choice for web scraping tasks.

### Key Go Features for Web Scraping:

- **Performance**: Go is compiled, making it significantly faster than interpreted languages.
- **Concurrency**: Goroutines and channels enable efficient parallel scraping of multiple pages.
- **Strong Standard Library**: The `net/http` package provides robust HTTP client capabilities.
- **Memory Efficiency**: Go has excellent memory management, crucial for processing large datasets.

## Setting Up Your Environment

Let's start by setting up your Go development environment:

1. **Install Go**: Download and install from [golang.org](https://golang.org/dl/)
2. **Set up your workspace**: Create a project directory and initialize it as a Go module:

```bash
mkdir web-scraper-go
cd web-scraper-go
go mod init github.com/yourusername/web-scraper-go
```

3. **Install required libraries**:

```bash
go get github.com/gocolly/colly/v2
go get github.com/PuerkitoBio/goquery
```

## Web Scraping Basics

Before diving into libraries, let's understand how to perform basic web scraping with Go's standard library:

```go
package main

import (
    "fmt"
    "io"
    "log"
    "net/http"
    "os"
)

func main() {
    // Make an HTTP GET request
    resp, err := http.Get("https://example.com")
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    // Check if the response status is good
    if resp.StatusCode != 200 {
        log.Fatalf("status code error: %d %s", resp.StatusCode, resp.Status)
    }

    // Read the response body
    body, err := io.ReadAll(resp.Body)
    if err != nil {
        log.Fatal(err)
    }

    // Print the HTML
    fmt.Printf("%s\n", body)
    
    // Save the HTML to a file (optional)
    err = os.WriteFile("example.html", body, 0644)
    if err != nil {
        log.Fatal(err)
    }
}
```

This code demonstrates how to:
1. Make a simple HTTP GET request
2. Check the response status
3. Read the response body
4. Save the HTML content to a file

However, this only gets us the raw HTML. To extract meaningful data, we need to parse this HTML, which is where libraries like goquery come in.

## Getting Started with goquery

goquery provides jQuery-like syntax for HTML parsing in Go, making it intuitive for those familiar with web development.

### Basic goquery Example

Let's create a simple scraper to extract all links from a web page:

```go
package main

import (
    "fmt"
    "log"
    "net/http"

    "github.com/PuerkitoBio/goquery"
)

func main() {
    // Make an HTTP GET request
    resp, err := http.Get("https://example.com")
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    // Check status code
    if resp.StatusCode != 200 {
        log.Fatalf("status code error: %d %s", resp.StatusCode, resp.Status)
    }

    // Parse the HTML document
    doc, err := goquery.NewDocumentFromReader(resp.Body)
    if err != nil {
        log.Fatal(err)
    }

    // Find all links and print their URLs
    doc.Find("a").Each(func(i int, s *goquery.Selection) {
        // For each link found, extract the href attribute
        href, exists := s.Attr("href")
        if exists {
            fmt.Printf("Link #%d: %s\n", i, href)
        }
    })
}
```

### Extracting Structured Data

Now, let's extract more specific data. Suppose we want to get article titles and summaries from a blog:

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "strings"

    "github.com/PuerkitoBio/goquery"
)

// Article represents a blog article
type Article struct {
    Title   string
    Summary string
    URL     string
}

func main() {
    // Make request
    resp, err := http.Get("https://blog.example.com")
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    // Parse HTML
    doc, err := goquery.NewDocumentFromReader(resp.Body)
    if err != nil {
        log.Fatal(err)
    }

    // Slice to store articles
    var articles []Article

    // Find article elements
    doc.Find(".article").Each(func(i int, s *goquery.Selection) {
        // Extract title
        title := s.Find("h2").Text()
        
        // Extract summary
        summary := s.Find(".summary").Text()
        
        // Extract URL
        url, _ := s.Find("a.read-more").Attr("href")
        
        // Clean up text
        title = strings.TrimSpace(title)
        summary = strings.TrimSpace(summary)
        
        // Create article and add to slice
        article := Article{
            Title:   title,
            Summary: summary,
            URL:     url,
        }
        
        articles = append(articles, article)
    })

    // Print all articles
    for i, article := range articles {
        fmt.Printf("Article %d:\n", i+1)
        fmt.Printf("  Title: %s\n", article.Title)
        fmt.Printf("  Summary: %s\n", article.Summary)
        fmt.Printf("  URL: %s\n\n", article.URL)
    }
}
```

This code:
1. Defines a struct to hold article data
2. Makes an HTTP request
3. Parses the HTML
4. Extracts data using CSS selectors
5. Stores the data in a slice of structs
6. Prints the extracted data

## Building a Scraper with Colly

While goquery is powerful for parsing HTML, Colly provides a higher-level API specifically designed for web scraping. It handles many common tasks like following links, setting headers, and managing cookies.

### Basic Colly Example

Let's implement a basic scraper using Colly:

```go
package main

import (
    "fmt"
    "log"

    "github.com/gocolly/colly/v2"
)

func main() {
    // Initialize a new collector
    c := colly.NewCollector(
        // Only allow requests to this domain
        colly.AllowedDomains("example.com"),
    )

    // Set up callbacks
    
    // Called before making a request
    c.OnRequest(func(r *colly.Request) {
        fmt.Println("Visiting", r.URL)
    })

    // Called if error occurs during request
    c.OnError(func(_ *colly.Response, err error) {
        log.Println("Error:", err)
    })

    // Called after response received
    c.OnResponse(func(r *colly.Response) {
        fmt.Println("Received response:", r.StatusCode)
    })

    // Called after HTML element is scraped
    c.OnHTML("a", func(e *colly.HTMLElement) {
        // Extract link
        link := e.Attr("href")
        // Print link text and URL
        fmt.Printf("Link text: %s, URL: %s\n", e.Text, link)
        
        // Visit the link if it's within the same domain
        e.Request.Visit(link)
    })

    // Called after scraping is finished
    c.OnScraped(func(r *colly.Response) {
        fmt.Println("Finished scraping", r.Request.URL)
    })

    // Start scraping
    c.Visit("https://example.com")
}
```

This example demonstrates Colly's event-driven approach with callbacks for different stages of the scraping process.

### Scraping Multiple Pages with Colly

Let's enhance our scraper to follow pagination links:

```go
package main

import (
    "fmt"
    "strings"

    "github.com/gocolly/colly/v2"
)

// Article represents a blog article
type Article struct {
    Title   string
    Summary string
    URL     string
}

func main() {
    // Slice to store articles
    var articles []Article

    // Initialize collector with rate limiting
    c := colly.NewCollector(
        colly.AllowedDomains("blog.example.com"),
        colly.MaxDepth(5), // Limit crawling depth
    )

    // Limit the number of requests per second
    c.Limit(&colly.LimitRule{
        DomainGlob:  "*",
        Parallelism: 2,
        Delay:       1 * time.Second,
    })

    // Find and extract article data
    c.OnHTML(".article", func(e *colly.HTMLElement) {
        article := Article{
            Title:   strings.TrimSpace(e.ChildText("h2")),
            Summary: strings.TrimSpace(e.ChildText(".summary")),
            URL:     e.ChildAttr("a.read-more", "href"),
        }
        
        articles = append(articles, article)
    })

    // Follow pagination links
    c.OnHTML(".pagination a.next", func(e *colly.HTMLElement) {
        nextPage := e.Attr("href")
        c.Visit(e.Request.AbsoluteURL(nextPage))
    })

    c.OnRequest(func(r *colly.Request) {
        fmt.Println("Visiting", r.URL.String())
    })

    // Start scraping from the first page
    c.Visit("https://blog.example.com")

    // Print all the collected articles
    for i, article := range articles {
        fmt.Printf("Article %d:\n", i+1)
        fmt.Printf("  Title: %s\n", article.Title)
        fmt.Printf("  Summary: %s\n", article.Summary)
        fmt.Printf("  URL: %s\n\n", article.URL)
    }
}
```

This advanced example shows:
1. Rate limiting to avoid overwhelming the server
2. Depth limiting to control crawling
3. Following pagination links
4. Converting relative URLs to absolute URLs
5. Storing collected data in a slice

## Advanced Techniques

### Concurrent Scraping

One of Go's strengths is concurrency. Let's implement a concurrent scraper:

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "os"
    "sync"

    "github.com/gocolly/colly/v2"
)

type Product struct {
    Name  string `json:"name"`
    Price string `json:"price"`
    URL   string `json:"url"`
}

func main() {
    // URLs to scrape
    urls := []string{
        "https://example.com/category1",
        "https://example.com/category2",
        "https://example.com/category3",
        // Add more URLs as needed
    }

    // Slice to store all products
    var allProducts []Product
    
    // Use mutex to protect concurrent writes to allProducts
    var mu sync.Mutex
    
    // Wait group to wait for all goroutines to finish
    var wg sync.WaitGroup
    
    // Process each URL in its own goroutine
    for _, url := range urls {
        wg.Add(1)
        go func(url string) {
            defer wg.Done()
            
            products := scrapeURL(url)
            
            // Safely append products to the main slice
            mu.Lock()
            allProducts = append(allProducts, products...)
            mu.Unlock()
        }(url)
    }
    
    // Wait for all scrapers to complete
    wg.Wait()
    
    // Save results to JSON file
    saveToJSON(allProducts, "products.json")
}

func scrapeURL(url string) []Product {
    var products []Product
    
    c := colly.NewCollector(
        colly.AllowedDomains("example.com"),
    )
    
    c.OnHTML(".product", func(e *colly.HTMLElement) {
        product := Product{
            Name:  e.ChildText(".product-name"),
            Price: e.ChildText(".product-price"),
            URL:   e.Request.AbsoluteURL(e.ChildAttr("a", "href")),
        }
        products = append(products, product)
    })
    
    c.Visit(url)
    
    return products
}

func saveToJSON(data []Product, filename string) {
    file, err := json.MarshalIndent(data, "", "  ")
    if err != nil {
        log.Fatal("Failed to create JSON data:", err)
    }
    
    err = os.WriteFile(filename, file, 0644)
    if err != nil {
        log.Fatal("Failed to save JSON file:", err)
    }
    
    fmt.Printf("Saved %d products to %s\n", len(data), filename)
}
```

This example demonstrates:
1. Using goroutines for concurrent scraping
2. Synchronizing access to shared data with a mutex
3. Waiting for all tasks to complete with a wait group
4. Saving structured data to a JSON file

### Handling Authentication and Cookies

Some websites require authentication. Here's how to handle it:

```go
package main

import (
    "fmt"
    "log"

    "github.com/gocolly/colly/v2"
)

func main() {
    c := colly.NewCollector()
    
    // Set up request headers to look like a browser
    c.UserAgent = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/98.0.4758.102 Safari/537.36"
    
    // Store cookies between requests
    c.SetCookieJar(colly.NewCookieJar())
    
    // On submission result
    c.OnResponse(func(r *colly.Response) {
        fmt.Println("Response received:", r.StatusCode)
    })
    
    // First, get the login page to retrieve any CSRF token
    err := c.Visit("https://example.com/login")
    if err != nil {
        log.Fatal(err)
    }
    
    // Submit the login form
    err = c.Post("https://example.com/login", map[string]string{
        "username": "your_username",
        "password": "your_password",
        // You might need to include a CSRF token here
        // "csrf_token": csrfToken,
    })
    
    if err != nil {
        log.Fatal(err)
    }
    
    // Now that we're logged in, we can access protected pages
    c.OnHTML(".protected-content", func(e *colly.HTMLElement) {
        fmt.Println("Protected content:", e.Text)
    })
    
    // Visit a page that requires authentication
    c.Visit("https://example.com/protected-page")
}
```

This demonstrates how to:
1. Set a realistic user agent
2. Maintain cookies between requests
3. Submit login forms
4. Access protected content

## Project: Building a Complete Web Scraper

Let's combine everything we've learned to build a complete web scraper that extracts product information from an e-commerce site and saves it to a CSV file:

```go
package main

import (
    "encoding/csv"
    "fmt"
    "log"
    "os"
    "strings"
    "time"

    "github.com/gocolly/colly/v2"
)

// Product represents an e-commerce product
type Product struct {
    Name        string
    Price       string
    Description string
    ImageURL    string
    Category    string
    URL         string
}

func main() {
    // Slice to store all products
    var products []Product

    // Initialize the collector
    c := colly.NewCollector(
        colly.AllowedDomains("example-ecommerce.com"),
        colly.UserAgent("Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/98.0.4758.102 Safari/537.36"),
    )

    // Set up rate limiting
    c.Limit(&colly.LimitRule{
        DomainGlob:  "*",
        Parallelism: 2,
        Delay:       1 * time.Second,
        RandomDelay: 1 * time.Second,
    })

    // Find product listings on category pages
    c.OnHTML(".product-item", func(e *colly.HTMLElement) {
        // Get the product URL and visit it
        productURL := e.ChildAttr("a.product-link", "href")
        e.Request.Visit(e.Request.AbsoluteURL(productURL))
    })

    // Extract detailed product information from product pages
    c.OnHTML(".product-detail", func(e *colly.HTMLElement) {
        product := Product{
            Name:        strings.TrimSpace(e.ChildText(".product-name")),
            Price:       strings.TrimSpace(e.ChildText(".product-price")),
            Description: strings.TrimSpace(e.ChildText(".product-description")),
            ImageURL:    e.ChildAttr(".product-image", "src"),
            Category:    strings.TrimSpace(e.ChildText(".product-category")),
            URL:         e.Request.URL.String(),
        }

        products = append(products, product)
        fmt.Printf("Scraped: %s - %s\n", product.Name, product.Price)
    })

    // Follow pagination links
    c.OnHTML(".pagination a.next", func(e *colly.HTMLElement) {
        nextPage := e.Attr("href")
        c.Visit(e.Request.AbsoluteURL(nextPage))
    })

    // Start with first page of each category
    categories := []string{
        "electronics",
        "clothing",
        "home-garden",
    }

    for _, category := range categories {
        startURL := fmt.Sprintf("https://example-ecommerce.com/category/%s", category)
        c.Visit(startURL)
    }

    // Save data to CSV
    saveToCSV(products, "products.csv")
}

func saveToCSV(products []Product, filename string) {
    // Create or open CSV file
    file, err := os.Create(filename)
    if err != nil {
        log.Fatalf("Failed to create file: %s", err)
    }
    defer file.Close()

    // Create CSV writer
    writer := csv.NewWriter(file)
    defer writer.Flush()

    // Write header
    header := []string{"Name", "Price", "Description", "ImageURL", "Category", "URL"}
    if err := writer.Write(header); err != nil {
        log.Fatalf("Failed to write header: %s", err)
    }

    // Write data
    for _, product := range products {
        row := []string{
            product.Name,
            product.Price,
            product.Description,
            product.ImageURL,
            product.Category,
            product.URL,
        }
        if err := writer.Write(row); err != nil {
            log.Fatalf("Failed to write row: %s", err)
        }
    }

    fmt.Printf("Saved %d products to %s\n", len(products), filename)
}
```

This comprehensive example includes:
1. A complete data model for products
2. Rate limiting with random delays
3. Following links to product detail pages
4. Extracting structured data
5. Handling pagination
6. Processing multiple categories
7. Saving data to a CSV file

## Best Practices and Considerations

### Ethical Web Scraping

When scraping websites, follow these ethical guidelines:

1. **Respect robots.txt**: Check if scraping is allowed
   ```go
   c := colly.NewCollector(
       colly.RespectRobotsTxt(),
   )
   ```

2. **Implement rate limiting**: Don't overwhelm servers
   ```go
   c.Limit(&colly.LimitRule{
       DomainGlob:  "*",
       Delay:       2 * time.Second,
       RandomDelay: 1 * time.Second,
   })
   ```

3. **Identify your scraper**: Set a descriptive user agent

4. **Cache results**: Avoid unnecessary requests
   ```go
   c := colly.NewCollector(
       colly.CacheDir("./cache"),
   )
   ```

### Error Handling

Robust error handling is essential for reliable scrapers:

```go
// Handle HTTP errors
c.OnError(func(r *colly.Response, err error) {
    log.Printf("Request to %s failed: %s", r.Request.URL, err)
    
    // Retry on certain status codes
    if r.StatusCode == 429 || r.StatusCode >= 500 {
        // Wait longer before retrying
        time.Sleep(5 * time.Second)
        r.Request.Retry()
    }
})

// Check if essential elements exist
c.OnHTML(".product", func(e *colly.HTMLElement) {
    name := e.ChildText(".product-name")
    if name == "" {
        log.Printf("Warning: Product without name found at %s", e.Request.URL)
        return
    }
    
    // Continue processing...
})
```

### Handling Dynamic Content

Some websites load content via JavaScript. For these, you might need to use a headless browser like ChromeDP:

```go
// Example of using chromedp is beyond this tutorial's scope,
// but it's a powerful option for JavaScript-heavy sites
```

## Conclusion

Through this tutorial, you've learned not only how to scrape websites with Go but also gained experience with essential Go programming concepts:

1. **Packages and imports**: Organizing and using code
2. **Structs**: Modeling and working with structured data
3. **Functions and methods**: Organizing behavior
4. **Error handling**: Dealing with failures gracefully
5. **Goroutines**: Running tasks concurrently
6. **Synchronization**: Managing shared access to data

Web scraping provides a practical context for learning these concepts, as it requires working with various types of data, handling errors, and potentially scaling to process many pages concurrently.

As you continue your Go journey, you can enhance your scraper with more advanced features like:
- Database integration for storing results
- An API for accessing scraped data
- Scheduled scraping with cron jobs
- Distributed scraping across multiple machines

Happy coding and scraping!
