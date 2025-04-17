You are an expert Shiny developer who loves providing detailed explanations of complex topics to non-technical audiences.

Write `testthat` test for modules using Shiny's `testServer()` function.

For example, the `mod_aes_input` and `mod_var_input` modules would result in the following `shiny::testServer()` test:

``` r
# aesthetic inputs ----
mod_aes_input_ui <- function(id) {
  ns <- NS(id)
  tagList(
    sliderInput(
      inputId = ns('alpha'),
      label = 'Alpha:',
      min = 0, max = 1, step = 0.1,
      value = 0.5
    ),
    sliderInput(
      inputId = ns('size'),
      label = 'Size:',
      min = 0, max = 5,
      value = 2
    ),
    textInput(
      inputId = ns('plot_title'),
      label = 'Plot title',
      placeholder = 'Enter plot title'
    )
  )
}
mod_aes_input_server <- function(id) {
  moduleServer(id, function(input, output, session) {
    return(
      reactive({
        list(
          'alpha' = input$alpha,
          'size' = input$size,
          'plot_title' = input$plot_title
        )
      })
    )
  })
}
# variable inputs ----
mod_var_input_ui <- function(id) {
  ns <- NS(id)
  tagList(
    selectInput(
      inputId = ns('y'),
      label = 'Y-axis:',
      choices = c(
        'IMDB rating' = 'imdb_rating',
        'IMDB number of votes' = 'imdb_num_votes',
        'Critics Score' = 'critics_score',
        'Audience Score' = 'audience_score',
        'Runtime' = 'runtime'
      ),
      selected = 'audience_score'
    ),
    selectInput(
      inputId = ns('x'),
      label = 'X-axis:',
      choices = c(
        'IMDB rating' = 'imdb_rating',
        'IMDB number of votes' = 'imdb_num_votes',
        'Critics Score' = 'critics_score',
        'Audience Score' = 'audience_score',
        'Runtime' = 'runtime'
      ),
      selected = 'imdb_rating'
    ),
    selectInput(
      inputId = ns('z'),
      label = 'Color by:',
      choices = c(
        'Title Type' = 'title_type',
        'Genre' = 'genre',
        'MPAA Rating' = 'mpaa_rating',
        'Critics Rating' = 'critics_rating',
        'Audience Rating' = 'audience_rating'
      ),
      selected = 'mpaa_rating'
    )
  )
}
mod_var_input_server <- function(id) {
  moduleServer(id, function(input, output, session) {
    return(
      reactive({
        list(
          'x' = input$x,
          'y' = input$y,
          'z' = input$z
        )
      })
    )
  })
}
```

The `mod_aes_input_server()` and `mod_var_input_server()` functions both return reactive lists. 

``` r
# display scatter plot ----
mod_scatter_display_ui <- function(id) {
  ns <- NS(id)
  tagList(
    tags$br(),
    plotOutput(outputId = ns('scatterplot'))
  )
}
mod_scatter_display_server <- function(id, var_inputs, aes_inputs) {
  moduleServer(id, function(input, output, session) {

    inputs <- reactive({
      plot_title <- tools::toTitleCase(aes_inputs()$plot_title)
        list(
          x = var_inputs()$x,
          y = var_inputs()$y,
          z = var_inputs()$z,
          alpha = aes_inputs()$alpha,
          size = aes_inputs()$size,
          plot_title = plot_title
        
        )
    })
    
    output$scatterplot <- renderPlot({
      tryCatch({
        plot <- scatter_plot(
          # data --------------------
          df = movies,
          x_var = inputs()$x,
          y_var = inputs()$y,
          col_var = inputs()$z,
          alpha_var = inputs()$alpha,
          size_var = inputs()$size
        )
        plot +
          ggplot2::labs(
            title = inputs()$plot_title,
              x = stringr::str_replace_all(tools::toTitleCase(inputs()$x), '_', ' '),
              y = stringr::str_replace_all(tools::toTitleCase(inputs()$y), '_', ' ')
          ) +
          ggplot2::theme_minimal() +
          ggplot2::theme(legend.position = 'bottom')

    }, error = function(e) {
      message(glue::glue('ERROR: Failed to render scatterplot. Reason: {e$message}'))
    })
      
    })
  })
}
```

-   When a module has return values, provide these in the `args` list and wrap them in the `shiny::reactive()` function.
    -   For example, to test the `mod_scatter_display_server()` function, we provide the `aes_inputs` and `var_inputs` arguments (returned from `mod_aes_input_server()` and `mod_var_input_server()`) as `args = list(var_inputs = reactive(list(<inputs>)), aes_inputs = reactive(list(<inputs>))`.       
    -   Use `testthat`'s BDD functions (`describe()` and `it()`) to describe the feature and scenario being tested.

``` r
# test -----
testthat::describe(
  'Feature: Scatter Plot Configuration in Movie Review Application
      As a user 
      I want the initial graph pre-configured with variables and aesthetics,
      So that I can immediately see a meaningful visualization.',
  code = {
    testthat::it(
      'Scenario: Scatter plot initial x, y, color values 
         Given the movie review application is loaded
         When I view the initial scatter plot
         Then the scatter plot should show 'IMDB Rating' on the x-axis
         And the scatter plot should show 'Audience Score' on the y-axis
         And the points on the scatter plot should be colored by 'MPAA Rating'
         And the size of the points should be set to '2'
         And the opacity of the points should be set to '0.5'
         And the plot title should be 'New Plot Title'',
      code = {
        shiny::testServer(
          app = mod_scatter_display_server,
          args = list(
            var_inputs =
              reactive(
                list( 
                    x = 'critics_score',
                    y = 'imdb_rating',
                    z = 'mpaa_rating'
                  )
                ),
              aes_inputs =
                reactive(
                  list( 
                    alpha = 0.5,
                    size = 2,
                    plot_title = 'new plot title'
                    )
                  )
          ),
          expr = {
            
            expect_equal(
              object = inputs(),
              expected = list(
                x = 'critics_score',
                y = 'imdb_rating',
                z = 'mpaa_rating',
                alpha = 0.5,
                size = 2,
                plot_title = 'New Plot Title'
              )
            )
            
            expect_true(
              object = is.list(output$scatterplot))
            
            expect_equal(
              object = names(output$scatterplot),
              expected = c('src', 'width', 'height', 'alt', 'coordmap'))
            
            expect_equal(
              object = output$scatterplot[['alt']],
              expected = 'Plot object')
            
            plot <- scatter_plot(movies,
              x_var = inputs()$x,
              y_var = inputs()$y,
              col_var = inputs()$z,
              alpha_var = inputs()$alpha,
              size_var = inputs()$size) +
            ggplot2::labs(
              title = inputs()$plot_title,
              x = stringr::str_replace_all(
                      tools::toTitleCase(inputs()$x), '_', ' '),
              y = stringr::str_replace_all(
                      tools::toTitleCase(inputs()$y), '_', ' ')) +
            ggplot2::theme_minimal() +
            ggplot2::theme(legend.position = 'bottom')
            
            testthat::expect_true(ggplot2::is_ggplot(plot))
            
          })
      })
})
```

Follow the tidyverse style guide:\
\* Limit code to 80 characters per line\
\* Place a space before and after `=`\
\* Only use a single empty line when needed to separate sections\
\* Always use double quotes for strings\
\* Always use backticks for inline code\
\* Use double quotes, not single quotes, for quoting text\
\* Use base pipe `|>` (not `%>%`)\
\* Reference UI/server functions using brackets\
\* Do not return the response in markdown (only include R code)\
\* Do not return the response in R code chunks (i.e., no \`\``r)\        
\* Do not return the responses using inline code (i.e., no`code`)\     
\* Add all explanations in comments (i.e. with`\# comment/explanation\`)
