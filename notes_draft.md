Chapter 1: Creating our own renderer
  - Goal: Modify the render method to to accept :pdf as an option and return a PDF created with Prawn

  1.1) Creating a rails plugin
    - rails plugin new plugin_name
    - generated files:
      - pdf_renderer.gemspec
        - provides basic gem specification
        - declares gem authors, version, dependencies, source files, etc.
        - same name as file inside the lib/ directory
      - Gemfile
        - Reuses gemspec dependencies
        - Also contains extra dependencies
      - Rakefile
        - Provides basic tasks to run the test suite, generate docs, and release the gem
    - Booting the dummy app
      - config/boot.rb
        - configure applications load path
        - points to Gemfile at the root of the plugin
        - Explicitly adds plugin's lib/ directory to Ruby's load path (so it's available inside the dummy app)
      - config/application.rb
        - stripped-down version of config/application.rb found in Rails apps
      - config/environment.rb
        - exactly the same as the one you'd find in a rails app

  1.2) Writing the Renderer
    - renderer: a hook exposed by the render method to customize it's behavior
    - Prawn: PDF-writing library for Ruby
      - add it to pdf_renderer.gemspec

  1.3) Understanding the Rails Rendering Stack
    - AbstractController
      - Centralizes common features of ActionCOntroller and ActionMailer
      - designed so developers can cherry-pick the functionality they want
      - rendering stack is responsible for normalizing arguments and converting them to a hash of options
        that ActionView::Renderer accepts (and then renders the template)

    - Text visualization of the rendering stack when render() is called with AbstractController::Rendering
      - render (AbstractController::Rendering)
        - _normalize_render (AbstractController::Rendering)
          - _normalize_args (AbstractController::Rendering)
              - converts the user-provided arguments into a hash
              - allows render(:new) to be converted into render(action: "new")
          - _normalize_options (AbstractController::Rendering)
            - further normalizes the hash that _normalize_args() returns
            - converts render(partial: true) to render(partial: action_name)
              - e.g. render(partial: "show")

        - render_to_body (AbstractController::Rendering)
          - _process_options (AbstractController::Rendering)
            - Processes all options that are meaningless to the view
            - e.g. status: 401
          - render_template (AbstractController::Rendering)
            - render (ActionView::Renderer)
              - called with two arguments
                1) view context
                  - instance of ActionView::Base
                  - the context in which our templates are evaluated
                  - receives view_assigns as an argument
                    - assigns: group of controller variables that will be accessible in the view
                2) hash of normalized options

    - Rails 2.3 vs 3.0
      - 2.3: View is responsible from retrieving assigns (controller) variables from the controller
      - 3.0+ Controller tells view which assigns to use

    - AbstractController::Rendering with AbstractController::Layouts
      - overrides _normalize_optoins to support the :layout option

    - AbstractController with ActionController (4 main modules)
      1) ActionController::Rendering
        - Overrides render() to check if it's ever called twice, raising DoubleRenderError if so
        - Overrides _process_options() to handle options such as :location, :status, :content_type
      2) ActionController::Renderers
        - Adds the API that allows for triggering of specific behavior whenever a given key is supplied
      3) ActionController::Instrumentation
        - Overloads render() so it can measure the time spent in the rendering stack
      4) ActionController::Streaming
        - Overloads _process_options() to handle the :stream by setting proper HTTP headers
        - overloads _render_template() to allow templates to be streamed

    - render() vs render_to_string()
      - render_to_string() does not store the rendered template as the response_body
      - some ActionController methods overload render() to add behavior while leaving render_to_string() alone (and vice-versa)

