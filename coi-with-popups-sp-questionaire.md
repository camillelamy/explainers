# COOP: same-origin-allow-popups-plus-coep

## Self-Review Questionnaire: Security and Privacy

#### What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

This feature allows the use of APIs gated behind crossOriginIsolation (SharedArrayBuffers) on pages that have COEP and COOP same-origin-allow-popups. Previously, those were only available to pages with COOP same-origin.

#### Do features in your specification expose the minimum amount of information necessary to enable their intended uses?

Yes.

#### How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?

The feature does not deal with PII.

#### How do the features in your specification deal with sensitive information?

The feature does not deal with sensitive information.

#### Do the features in your specification introduce new state for an origin that persists across browsing sessions?

No.

#### Do the features in your specification expose information about the underlying platform to origins?

No.

#### Does this specification allow an origin to send data to the underlying platform?

No.

#### Do features in this specification allow an origin access to sensors on a user’s device?

No.

#### What data do the features in this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.

This feature allows the use of APIs gated behind crossOriginIsolation (SharedArrayBuffers) on pages that have COEP and COOP same-origin-allow-popups. Previously, those were only available to pages with COOP same-origin.

#### Do features in this specification enable new script execution/loading mechanisms?

No.

#### Do features in this specification allow an origin to access other devices?

No.

#### Do features in this specification allow an origin some measure of control over a user agent’s native UI?

No.

#### What temporary identifiers do the feautures in this specification create or expose to the web?

None.

#### How does this specification distinguish between behavior in first-party and third-party contexts?

COOP is only applicable to top-level frame, so the notion of first-party and third-party contexts does not apply here.

#### How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

No difference with regular mode.

#### Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

Yes.

#### Do features in your specification enable origins to downgrade default security protections?

No.

