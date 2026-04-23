# fixtures/

Empty in v1.0.0.

Test fixtures (valid and invalid manifests, example completion files, a smoke-test WAV, example `runtime/status.json` snapshots) are populated in **Step 5** of the action plan — after Phase 2's minimal `noted` runtime is ready to exercise them.

When filled, the target layout is roughly:

```
fixtures/
  manifests/
    valid/
      calendar_in_person.json
      calendar_online.json
      calendar_with_next_meeting.json
      adhoc.json
    invalid/
      missing_required_fields.json
      wrong_asr_backend.json
      wrong_major_version.json
  completion/
    completed.json
    completed_with_warnings.json
    failed_startup.json
  runtime/
    recording.json
    processing.json
  audio/
    smoke_test.wav
```

Additions under this directory do not require a version bump as long as they satisfy the current schemas. Changes that alter fixture **semantics** warrant a patch bump — see `versioning-policy.md`.
