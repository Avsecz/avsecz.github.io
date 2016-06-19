options(repos=structure(c(CRAN="http://ftp5.gwdg.de/pub/misc/cran/")))
options(browser = "google-chrome")

library(knitr)

## key-binded to C-c C-s
render_pdf <- function(f){
    rmarkdown::render(f,output_format="pdf_document")
}

## for emacs
rmarkdown_open_chrome <- function(input, ...){
    ## substitute .R with .html
    html_input <- gsub("\\.R$","\\.html",input)
    browseURL(html_input, browser="google-chrome")
}

##' knit all posts
##' place your .Rmd or .R files into:
##' _Rmd folder
##' @param site.path Root directory of your blog
##' @param overwriteAll Weather to rebuild all the posts
##' @param overwriteOne Regex for files to recompile.
##' @param ... Additional options to opts_chunk$set
knit_post <- function(site.path=getwd(), overwriteAll=F, overwriteOne=NULL,
                      ...
                      ) {
  if(!'package:knitr' %in% search()) library('knitr')
  
  ## Blog-specific directories.  This will depend on how you organize your blog.
  site.path <- site.path # directory of jekyll blog (including trailing slash)
  rmd.path <- file.path(site.path, "_Rmd") # directory where your Rmd-files reside (relative to base)
  fig.dir <- "images/Rmd/" # directory to save figures
  posts.path <- file.path(site.path, "_posts/") # directory for converted markdown files
  cache.path <- file.path(site.path, "_cache") # necessary for plots
  
  render_jekyll()
  opts_knit$set(base.url = '/', base.dir = site.path)
  opts_chunk$set(fig.path=fig.dir,
                 ## fig.width=8.5,
                 ## fig.height=5.25,
                 ## dev='svg',
                 ## cache
                 cache=F,
                 cache.path=cache.path,
                 warning=F, message=F ,
                 tidy=F,
                 ...
                 )
  

  ## setwd(rmd.path) # setwd to base
  
  # some logic to help us avoid overwriting already existing md files
  files.rmd <- data.frame(rmd = list.files(path = rmd.path,
                                full.names = T,
                                pattern = "\\.Rmd$|\\.R$", #search  only for .Rmd file
                                ignore.case = T,
                                recursive = F), stringsAsFactors=F)
  files.rmd$corresponding.md.file <- paste0(posts.path, "/", basename(gsub(pattern = "\\.Rmd$|\\.R$", replacement = ".md", x = files.rmd$rmd)))
  files.rmd$corresponding.md.exists <- file.exists(files.rmd$corresponding.md.file)
  
  ## determining which posts to overwrite from parameters overwriteOne & overwriteAll
  files.rmd$md.overwriteAll <- overwriteAll
  if(is.null(overwriteOne)==F) files.rmd$md.overwriteAll[grep(overwriteOne, files.rmd[,'rmd'], ignore.case=T)] <- T
  files.rmd$md.render <- F

  for (i in 1:dim(files.rmd)[1]) {
    if (files.rmd$corresponding.md.exists[i] == F) {
      files.rmd$md.render[i] <- T
    }
    if ((files.rmd$corresponding.md.exists[i] == T) && (files.rmd$md.overwriteAll[i] == T)) {
      files.rmd$md.render[i] <- T
    }
  }
  
  # For each Rmd file, render markdown (contingent on the flags set above)

  for (i in 1:nrow(files.rmd)) {
    if (files.rmd$md.render[i] == T) {
      render_spin <- FALSE

      ## if the file is .R, convert it to a .Rmd
      if (grepl("\\.R$", files.rmd$rmd[i])) {
        render_spin <- TRUE
        spin(files.rmd$rmd[i], knit = FALSE,
             format = "Rmd"
             )
        files.rmd$rmd[i] <- paste0(files.rmd$rmd[i], "md")
      }
      
      out.file <- knit(input = as.character(files.rmd$rmd[i]), 
                      output = as.character(files.rmd$corresponding.md.file[i]),
                      envir = parent.frame(),
                      quiet = F)
      message(paste0("KnitPost(): ", basename(files.rmd$rmd[i])))

      ## if we converted .R, remove the intermediate .Rmd file
      if (render_spin &
            ## both files exist and the file-name is indeed .Rmd
            file.exists(files.rmd$rmd[i]) &
            grepl("\\.Rmd$",files.rmd$rmd[i]) & 
            file.exists(gsub("\\.Rmd", ".R", files.rmd$rmd[i]))
          ) {
        
        file.remove(files.rmd$rmd[i])
      }

      
    }     
  }
  
}
