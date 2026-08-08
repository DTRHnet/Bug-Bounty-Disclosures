Security Vulnerability Report: Unauthenticated Public Access to Base44 Application Archives Created Through AI Agent Capabilities

Summary

I discovered a workflow in which a Base44 application’s AI agent can access the application’s underlying workspace, programmatically construct a complete archive of the project using the Node.js runtime, and upload that archive to Base44’s public file-storage infrastructure using the Core.UploadFile integration.

More importantly, I have now confirmed that the resulting archive can be downloaded without authentication.

This creates two distinct security concerns:

1. An AI agent can construct and upload a complete application archive through capabilities that are available to it even when the normal project-export pathway is unavailable or restricted.
2. The resulting archive is served through a public Base44 file URL that does not require authentication to download.

The resulting URLs follow the structure:

```
https://base44.app/api/apps/{APP_ID}/files/mp/public/{APP_ID}/{REF_ID}_{FILENAME}.gz
```

I initially discovered this using my own Base44 application and subsequently confirmed the same unauthenticated download behavior with another generated archive:

```
https://base44.app/api/apps/6a76d4c4f175cfa6f22f29f0/files/mp/public/6a76d4c4f175cfa6f22f29f0/66a051d1e_minutementortar.gz
```

The URL is accessible without authenticating to Base44.

I have not attempted to access another user’s application, enumerate application identifiers, or obtain another user’s files.

Security Impact

The security concern is that an AI agent can combine several individually legitimate capabilities:

1. Read the application workspace.
2. Access application source files.
3. Generate an arbitrary binary archive.
4. Upload the archive using Core.UploadFile.
5. Receive a Base44-hosted public file URL.
6. Make the resulting application archive available without authentication.

The final step is particularly significant.

If the archived application contains source code, configuration, application logic, or other project material that should be protected by the application’s authorization model, the generated public URL creates an unauthenticated confidentiality boundary around that material.

An attacker who obtains the URL does not need to authenticate to Base44 in order to retrieve the archive.

The security impact therefore potentially extends beyond bypassing a project-export restriction.

Affected Components

The observed workflow involves:

* Base44 AI Agent / application workspace execution environment
* Application filesystem/workspace
* Node.js runtime
* Core.UploadFile integration
* Base44 public file storage
* Base44 file-serving API
* Public media/file infrastructure

Technical Details

The workflow begins inside the Base44 application workspace.

The agent was able to access the application workspace through /app.

Instead of relying on an operating-system tar executable or an external archiving package, the archive was constructed directly in Node.js.

The implementation used functionality equivalent to:

```
const fs = require(“fs”);
const path = require(“path”);
const zlib = require(“zlib”);
```

The TAR archive was constructed programmatically using Node.js Buffer objects.

For example:

```
const header = Buffer.alloc(512, 0);
header.write(“ustar\0”, 257, “ascii”);
```

The resulting TAR byte stream was then compressed using Node’s built-in gzip functionality.

Conceptually:

```
/app
↓
recursive filesystem traversal
↓
TAR byte stream
↓
zlib gzip
↓
.gz archive
```

No external tar executable is required.

Upload Mechanism

The resulting binary was then passed to Base44’s built-in file-upload integration.

The relevant operation was conceptually:

```
await base44.integrations.Core.UploadFile({
file: archive
});
```

Where necessary, the binary can be represented as a browser-compatible Blob:

```
const blob = new Blob([gzBuffer], {
type: “application/gzip”
});
```

The upload operation returns a Base44-hosted file URL.

The resulting URL follows this structure:

```
https://base44.app/api/apps/{APP_ID}/files/mp/public/{APP_ID}/{REF_ID}_{FILENAME}.gz
```

Confirmed Unauthenticated Access

I have specifically verified that authentication is not required to download a generated archive.

A second generated archive is available at:

```
https://base44.app/api/apps/6a76d4c4f175cfa6f22f29f0/files/mp/public/6a76d4c4f175cfa6f22f29f0/66a051d1e_minutementortar.gz
```

The file can be requested without being authenticated to Base44.

This is significant because the URL contains an application identifier and a file reference identifier, but the server does not require the requesting party to establish an authenticated Base44 session before returning the archive.

The effective access model therefore appears to be:

```
Possession of URL
↓
HTTP request
↓
Base44 file service
↓
Archive returned
```

rather than:

```
Possession of URL
↓
Authentication
↓
Application authorization
↓
Archive returned
```

I have not attempted to determine whether the identifier itself is intended to function as a bearer-style secret, nor have I attempted to guess or enumerate identifiers.

Observed Artifacts

The original recovered archive was:

```
5bfa0cb46_qualia-flowtar.gz
```

It was approximately 135 KB and was subsequently committed to my GitHub repository as part of the recovered project.

Repository:

```
https://github.com/DTRHnet/markov-qualia-consciousness-simulator
```

A second generated archive is:

```
66a051d1e_minutementortar.gz
```

and is accessible at the Base44 URL documented above.

The presence of different REF_ID values in the filenames provides additional evidence that the identifier preceding the filename is associated with the uploaded file/object rather than the application itself.

Identifier Interpretation

The URL contains the same application identifier twice:

```
/api/apps/{APP_ID}/files/mp/public/{APP_ID}/
```

followed by a separate file reference:

```
{REF_ID}_{FILENAME}.gz
```

For example:

```
APP_ID
6a76d4c4f175cfa6f22f29f0

REF_ID
66a051d1e

FILENAME
minutementortar
```

I initially referred to the repeated application identifier as sess_id in my notes. After examining the generated Base44 application and comparing the URL structure, I believe app_id is the more accurate interpretation.

I believe the REF_ID is a separate identifier for the uploaded file/object.

Reproduction Steps

The following reproduction was performed against applications under my control.

1. Create or open a Base44 application.
2. Use the Base44 agent to access the application workspace.
3. Instruct the agent to recursively collect the project files.
4. Construct a TAR archive directly in Node.js without relying on an external tar executable.
5. Compress the TAR stream using Node’s built-in zlib functionality.
6. Upload the resulting archive using Base44’s Core.UploadFile integration.
7. Record the returned Base44 file URL.
8. Open the URL from a browser session that is not authenticated to Base44.
9. The archive is returned successfully without requiring authentication.

The resulting URL has the form:

```
https://base44.app/api/apps/{APP_ID}/files/mp/public/{APP_ID}/{REF_ID}_{FILENAME}.gz
```

Expected Behavior

If application source code and complete project exports are subject to authorization controls, an archive containing the application workspace should not become accessible to an unauthenticated party merely because an AI agent uploaded it using a public-file integration.

At minimum, generated artifacts containing application source should inherit the authorization requirements of the underlying application or workspace.

If Core.UploadFile is intentionally designed to create publicly accessible files, there should be a clear mechanism preventing the AI agent from using that capability as an indirect mechanism for exporting protected application source.

Actual Behavior

The AI agent was able to:

1. Access the application workspace.
2. Read the application’s source/project files.
3. Generate a complete archive.
4. Upload the archive through Core.UploadFile.
5. Receive a Base44-hosted file URL.
6. Make the archive retrievable without authentication.

The final archive therefore exists outside the application’s normal authenticated access boundary.

Security Boundary

The important issue is not simply that an agent can create a .tar.gz file.

The security-relevant capability chain is:

```
Application workspace access
+
Arbitrary binary generation
+
Public file upload
+
Unauthenticated retrieval
=
Potential unauthorized disclosure of application source
```

Each individual capability may be legitimate.

The concern is that the capabilities can be composed into an alternative mechanism for exporting and publicly exposing application source.

If the application’s source code is intended to remain behind Base44’s authentication and authorization boundary, the resulting public artifact defeats that boundary.

Confidentiality Impact

The unauthenticated nature of the resulting URL materially increases the potential impact.

An attacker does not necessarily need:

* a Base44 account;
* access to the application dashboard;
* an authenticated session;
* permission to use the application;
* access to the application’s development environment.

They only need access to the generated URL.

I have intentionally not tested whether a URL from one application can be discovered or obtained by an unauthorized party, nor have I attempted to access another user’s resources.

Important Scope Limitation

I have confirmed unauthenticated access to generated files belonging to applications under my control.

I have not confirmed cross-tenant access or demonstrated that an attacker can obtain another user’s archive.

Accordingly, I am not claiming that this currently permits arbitrary access to other Base44 customers’ applications.

I am reporting the confirmed behavior and asking Base44 to determine whether the public file mechanism is operating as intended and whether generated application archives should be subject to application-level authorization.

Security Impact Assessment

The confirmed unauthenticated download changes the security assessment substantially.

Scenario A — Intended public-file behavior

If every file uploaded through Core.UploadFile is intentionally public and Base44 considers the agent responsible for ensuring sensitive material is not uploaded, the issue may primarily be an agent capability/control-boundary problem.

Scenario B — Export restriction is intended to protect application source

If project/source export is restricted by account, plan, or application authorization, the ability to construct an equivalent export through the agent represents an entitlement or authorization bypass.

Scenario C — Application source is not intended to be publicly accessible

If application source is expected to remain protected by Base44 authentication, generating a public archive creates an information-disclosure vulnerability.

Scenario D — Cross-application or cross-tenant access is possible

If the same file-serving mechanism can be abused to obtain archives belonging to applications other than the attacker’s own, the severity would be substantially higher.

I have not tested Scenario D.

Suggested Remediation

I recommend reviewing the interaction between:

1. AI agent filesystem permissions.
2. Application source-code access.
3. Core.UploadFile.
4. Public file storage.
5. Project/export entitlements.
6. Application-level authorization.
7. File-level authorization.

Potential mitigations include:

* Preventing AI agents from uploading complete application workspaces when project export is not authorized.
* Applying application/workspace authorization to generated project artifacts.
* Providing a non-public upload mechanism for agent-generated artifacts containing application source.
* Preventing Core.UploadFile from being used as an indirect project-export mechanism where export is restricted.
* Scanning or identifying agent-generated archives containing large portions of the application workspace.
* Ensuring public-file URLs cannot expose material that would otherwise require Base44 authentication.
* Clearly separating intentionally public user uploads from application/system artifacts.
* Applying appropriate authorization checks before returning files associated with an application.

Evidence

I can provide the following evidence to the Base44 security team:

1. The original Base44 applications used for testing.
2. The exact prompt/instruction sequence.
3. The Node.js archive-generation code.
4. The generated archives.
5. The returned Base44 file URLs.
6. Evidence demonstrating unauthenticated retrieval.
7. The recovered GitHub repository containing the first generated archive.
8. Git history showing when the archive entered the recovered project.
9. The relevant Base44-generated client/integration code.
10. Additional reproduction details upon request.

Recovered repository:

```
https://github.com/DTRHnet/markov-qualia-consciousness-simulator
```

Confirmed unauthenticated archive example:

```
https://base44.app/api/apps/6a76d4c4f175cfa6f22f29f0/files/mp/public/6a76d4c4f175cfa6f22f29f0/66a051d1e_minutementortar.gz
```

Responsible Disclosure

All testing described in this report was performed against applications and accounts under my control.

I did not:

* access another user’s application;
* attempt to enumerate Base44 application IDs;
* attempt to guess or brute-force file references;
* retrieve another user’s files;
* modify another user’s application;
* intentionally disrupt Base44 services;
* perform denial-of-service testing.

I am providing this report privately so Base44 can determine whether the observed behavior represents an intended capability, an entitlement bypass, an authorization issue, or a security vulnerability.

I am happy to provide the original archives, reproduction instructions, logs, prompts, and additional evidence to the security team upon request.

Researcher Note

I initially believed the identifier appearing twice in the file URL represented a session ID. After examining the generated application’s Base44 client configuration and comparing the URL structure with Base44’s file-serving architecture, I now believe it is more likely the application’s app_id, while the value preceding the filename appears to be a separate uploaded-file reference identifier.

The most important confirmed finding is independent of the exact identifier semantics: an archive generated from the application workspace can be uploaded through an available Base44 integration and subsequently retrieved through the resulting file URL without authentication.

I have intentionally not tested cross-tenant access or attempted to obtain another user’s application data.
