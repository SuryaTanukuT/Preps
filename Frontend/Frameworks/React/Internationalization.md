# Internationalization (i18n) in React (interview + practical)
Internationalization (i18n) = building your app so it can support multiple languages/locales.Localization (l10n) = adding the actual translations and locale-specific formatting.
Internationalization in React means designing the app to support multiple locales: translating UI text via stable keys, handling pluralization properly, formatting dates/numbers/currencies using Intl, and supporting RTL layouts. I typically use a library like react-i18next with namespaces and lazy-loaded locale bundles, keep locale in global state or URL, provide fallbacks, and test long strings and RTL to avoid layout issues.

# Most common libraries (interview) i18next + react-i18next (most used)Pros: powerful, pluralization, namespaces, lazy loading, SSR supportCons: setup overhead
 FormatJS / react-intlPros: strong ICU message formattingCons: more verbose for some teams
If asked “what would you choose?”:“react-i18next is common for product apps; react-intl is great when you want ICU messages and formatting-heavy strings.”