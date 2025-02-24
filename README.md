# Case Study - Foreign Languages known 

## The Task
The goal of this Case Study is to analyse the foreign language proficiency of individuals living in Germany, France, Belgium and Sweden compared to the EU average. The focus of this Analysis lies on the number of foreign languages known by individuals aged 18-69 for the year 2022. The Dataset is from [Eurostat](https://ec.europa.eu/eurostat/en/).

## The Graph
![](CaseStudy_Graph.png)

Source:
[Eurostat](https://ec.europa.eu/eurostat/databrowser/view/edat_aes_l23/default/table?lang=en&category=educ.educ_lang.educ_lang_00.edat_aes_l2)

## The R Code

    library(dplyr)

    ## 
    ## Attache Paket: 'dplyr'

    ## Die folgenden Objekte sind maskiert von 'package:stats':
    ## 
    ##     filter, lag

    ## Die folgenden Objekte sind maskiert von 'package:base':
    ## 
    ##     intersect, setdiff, setequal, union

    library(ggplot2)
    library(ggtext)
    library(countrycode)

    # set wd to script's source
    setwd(dirname(rstudioapi::getActiveDocumentContext()$path))

    # load dataset
    data <- read.csv("lang_known.csv")

    ###########################
    # Data Manipulation
    ###########################

    selected <- data %>%
      # filter by year 2022
      filter(TIME_PERIOD == 2022) %>%
      #filter for Germany, EU, France, Belgium and Sweden
      filter(geo %in% c("European Union - 27 countries (from 2020)", "Germany", "France", "Belgium", "Sweden")) %>%
      # filter for individuals aged 18–69 years
      filter(age == "From 18 to 69 years") %>%
      # convert n_lang into numbers
      mutate(n_lang = case_when(
        n_lang == "No languages" ~ "0",
        n_lang == "1 language" ~ "1",
        n_lang == "2 languages" ~ "2",
        n_lang == "3 languages or more" ~ "3+",
      )) %>%
      # filter for all educational backgrounds
      filter(isced11 == "All ISCED 2011 levels") %>%
      # separate EU data to calculate difference in avg's
      group_by(n_lang) %>%
      mutate(EU_Avg = OBS_VALUE[geo == "European Union - 27 countries (from 2020)"]) %>%
      ungroup() %>%
      # Compute the difference between country average and EU average
      mutate(avg_diff = EU_Avg - OBS_VALUE) %>%
      # Keep only one row per country and known language
      distinct(geo, n_lang, .keep_all = TRUE) %>%
      # drop unused attributes
      select( -c(DATAFLOW, LAST.UPDATE, freq, unit, OBS_FLAG)) %>%
      # drop the EU data
      filter(geo != "European Union - 27 countries (from 2020)")

    ###########################
    # Data Visualization         
    ###########################

    selected %>%
      mutate(
        # get iso 2-letter code dynamically
        iso2 = countrycode(geo, origin = "country.name", destination = "iso2c"),
        # create geo_label with flags and country name
        geo_label = paste0(
          "<img src='https://flagcdn.com/w40/", tolower(iso2), ".png' height='15' width='25'/> ",
          geo, 
          " - <img src='https://flagcdn.com/w40/eu.png' width='25'/>"
        )
      ) %>%
      ggplot(aes(x = as.factor(n_lang), y = avg_diff, fill = avg_diff)) + 
      
      # add barplot
      geom_bar(stat = "identity", width = 0.7, aes(alpha = abs(avg_diff)), color = "black") +
      
      # Add dotted horizontal line at 0%
      geom_hline(yintercept = 0, linetype = 'dotted', color = 'black') +
      
      # Add values to the bars
      geom_text(
        aes(label = round(avg_diff, 1)), 
        vjust = ifelse(selected$avg_diff > 0, -0.5, 1.5),  # Above for positive, below for negative
        size = 3, color = "black"
      ) +
      
      # Custom gradient scale for fill: Darker shades of blue for low negative values and red for positive
      scale_fill_gradientn(
        colours = c("blue3", "blue", "white", "red", "red3")
      ) +
      
      # Set the alpha range to ensure the transparency works correctly
      scale_alpha_continuous(range = c(0.3, 1)) +  # Lower transparency for smaller values and full opacity for larger ones
      
      # Add custom scale for y-axis
      scale_y_continuous(
        breaks = seq(-20, 25, by = 5),
        limits = c(-20, 25)
      ) +
      
      # Facet by geo_label with free scales
      facet_wrap(~ geo_label, labeller = label_value, scales = "free") +
      
      # Add labels and theme customization
      labs(
        title = "Foreign Language Proficiency by Country: Comparison with EU Average",
        x = "Number of Foreign Languages",
        y = "Difference in Percentage"
      ) +
      theme_minimal() +
      theme(
        panel.border = element_rect(fill = "transparent", color = "black", linewidth = 1),
        plot.title = element_text(face = "bold", hjust = 0.5),
        strip.text = ggtext::element_markdown(size = 12, face = "bold"),
        axis.title.x = element_text(size = 10, face = "bold"),
        axis.title.y = element_text(size = 10, face = "bold"),
        legend.position = "none"
      )
