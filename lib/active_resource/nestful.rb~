gem "nestful"

require "nestful"
require "active_resource"

module ActiveResource
  module Nestful
    class Base < ActiveResource::Base
      class << self

        def oauth(options = nil)
          @oauth = options if options

          if block_given?
            yield
            @oauth = nil
          end

          if defined?(@oauth)
            @oauth
          elsif superclass != Object &&
                  superclass != ActiveResource::Base &&
                    superclass.oauth
            superclass.oauth.dup.freeze
          end
        end

        alias_method :oauth=, :oauth

        def format=(mime_type_reference_or_format)
          format = mime_type_reference_or_format.is_a?(Symbol) ?
            ::Nestful::Formats[mime_type_reference_or_format] : mime_type_reference_or_format

          format = format.new

          write_inheritable_attribute(:format, format)
          connection.format = format if site
        end

        def format
          read_inheritable_attribute(:format) || ::Nestful::Formats::XmlFormat.new
        end

        def connection(refresh = false)
          return @connection if @connection
          @connection = ActiveResource::Nestful::Connection.new(site, format) if refresh || @connection.nil?
          @connection.proxy = proxy if proxy
          @connection.user = user if user
          @connection.password = password if password
          @connection.auth_type = auth_type if auth_type
          @connection.timeout = timeout if timeout
          @connection.ssl_options = ssl_options if ssl_options
          @connection.oauth = oauth if oauth
          @connection
        end
      end

      def to_multipart_form(options={})
        existing_format = self.class.format
        self.format = :multipart unless existing_format.is_a? ::Nestful::Formats[:multipart]
        self.class.format.encode({
          self.class.element_name => serializable_hash(options)
        })
      ensure
        self.format = existing_format
      end

      def save(options=nil)
        super
      rescue ::Nestful::ResourceInvalid => e
        # cache the remote errors because every call to <tt>valid?</tt> clears
        # all errors. We must keep a copy to add these back after local
        # validations
        @remote_errors = e
        load_remote_errors(@remote_errors, true)
        false
      end

      def load_remote_errors(remote_errors, save_cache = false)
        case self.class.format
        when ::Nestful::Formats[:xml]
          errors.from_xml(remote_errors.response.body, save_cache)
        when ::Nestful::Formats[:json]
          errors.from_json(remote_errors.response.body, save_cache)
        when ::Nestful::Formats[:multipart]
          case self.class.format.decode_format
          when :xml
            errors.from_xml(remote_errors.response.body, save_cache)
          when :json
            errors.from_json(remote_errors.response.body, save_cache)
          else
            super
          end
        else
          super
        end
      end
    end # Base
  end # Nestful
end # ActiveResource

# NOTE: ActiveResource uses 422 to throw errors on object create / update
# and these errors are json in the body content. We need to make sure that
# we read the body content of any 422 errors. Easiest place to do this
# given nestful's architecture is in the initialize of the 422 (ResourceInvalid)
# exception itself.
module ::Nestful
  class ResourceInvalid
    def initialize(response, message = nil)
      super
      @response.read_body
    end
  end
end

require File.join(File.dirname(__FILE__), *%w[nestful connection])
require File.join(File.dirname(__FILE__), *%w[nestful formats multipart])