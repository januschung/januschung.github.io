# Helm Template Tips

Helm templates provide a powerful way to configure Kubernetes manifests dynamically. In this post, we’ll cover some useful tricks, including:

- Handling optional maps

- Setting default values

- Using ternary expressions

- Other useful Helm template functions

![Helm Template Tips](../../assets/tech-blog/devops/helm-template-tips/banner.jpg)

## Handling Optional Maps

Sometimes, you want to define environment variables dynamically from an optional map. Here’s a Helm template snippet to achieve this:

``` groovy
{{- if .Values.env }}
env:
  {{- range $k, $v := .Values.env }}
  - name: {{ $k }}
    value: {{ $v }}
  {{- end }}
{{- end }}
```

### Example Values File:

``` yaml
env:
  NODE_ENV: production
  API_URL: https://api.example.com
```

If `.Values.env` is not provided, the `env` section is omitted entirely.

### Defining and Using a Map in the Template

Instead of defining `env` in `values.yaml`, you can construct it dynamically using `dict`:

``` groovy
{{- $env := dict "CYPRESS_APP_VERSION" .Values.global.app.version }}
```

Then, you can use it as follows:

``` groovy
{{- if $env }}
env:
  {{- range $k, $v := $env }}
  - name: {{ $k }}
    value: {{ $v }}
  {{- end }}
{{- end }}
```

## Setting Default Values

If a value is optional but you want to ensure a default is used, use `default`:

``` yaml
image:
  tag: "{{ .Values.image.tag | default "latest" }}"
```

This ensures `image.tag` is set to `latest` if not explicitly provided.

## Using Ternary Expressions

``` yaml
host: {{ eq .Values.preview "true" | ternary .Values.global.domain.previewHost .Values.global.domain.host }}
```

Here, if `.Values.preview` is `"true"`, `host` is set to `.Values.global.domain.previewHost`; otherwise, it is set to `.Values.global.domain.host`.

## Checking for Nested Values Safely

To avoid errors when accessing nested values, use `hasKey`:

``` groovy
{{- if hasKey .Values "database" }}
db:
  host: {{ .Values.database.host | default "localhost" }}
  port: {{ .Values.database.port | default 5432 }}
{{- end }}
```

This ensures `.Values.database` exists before accessing its fields.

## Combining Multiple Defaults

You can chain `default` functions:

``` groovy
appPort: {{ .Values.service.port | default .Values.global.defaultPort | default 8080 }}
```

This checks for `service.port`, then `global.defaultPort`, and falls back to `8080`.

## Conclusion

Helm templates provide flexible ways to handle optional values, defaults, and conditional logic. These tricks help create robust and reusable Helm charts.
