# Local patches

Fork of [oracle-rs 0.1.7](https://crates.io/crates/oracle-rs/0.1.7) (MIT OR Apache-2.0,
copyright Stian Grytøyr), used by DBX Lite. The first commit on this branch is the unmodified 0.1.7 package;
everything after it is a local patch. Keep this list current so the crate can be rebased onto a newer upstream.

1. `Config::sysdba(bool)` / `Config::sysdba` — authenticate with the SYSDBA privilege.
   Upstream implements `AuthMessage::with_sysdba` but never exposes it through `Config`.
   Touches `src/config.rs` and `AuthMessage` construction in `Connection::authenticate` (`src/connection.rs`).

2. `Connection::disconnected(Config)` — a never-connected `Connection` whose operations all fail, so callers
   can build client values in tests without a database. Touches `src/connection.rs`.

3. Marker packets (BREAK/RESET) use the 32-bit length field when a large SDU was negotiated
   (`ConnectionInner::send_marker`, `Connection::send_marker` in `src/connection.rs`). Upstream wrote a 16-bit
   length plus zeros, which Oracle 23ai reads as a bogus length and answers by closing the socket, so every
   ORA-error on a statement killed the session.
4. `ORACLE_RS_TRACE=1` prints every received packet header to stderr (debugging aid).

All of the following live in `src/connection.rs` unless noted. They were found against Oracle 23ai
(`gvenzl/oracle-free`, TTC field version >= 18) and follow python-oracledb's packet semantics.

5. Error-info trailer: the sql_type/checksum ub4 fields are skipped when the negotiated TTC field version is
   >= `FIELD_VERSION_20_1` (14, added to `src/constants.rs`). Without this every ORA message came back garbled.
   The version is stored in `ttc_field_version` after `negotiate_protocol`.
6. `parse_error_message_info` skips the rowid as ub fields instead of misreading them.
7. `has_more_rows` is `error_code == 0` in both the query and fetch response parsers (upstream returned a
   fixed value, which truncated paged results at the first batch or raised ORA-01002).
8. Fetch requests use the large-SDU framing, sequence number and, for field version >= 18, the token
   (`src/messages/fetch.rs`: `build_request_with_sdu`). `fetch_more` also handles a MARKER by resetting.
9. Server-side piggyback messages (type 23) are skipped in the DML, query and fetch response loops
   (`skip_server_side_piggyback`); return-parameter parsing reads a ub2 length before each key/value.
10. Describe info captures `csfrm` and `nullable`. NCHAR/NVARCHAR decode as UTF-16BE; BINARY_FLOAT/DOUBLE,
    BOOLEAN, INTERVAL YEAR TO MONTH / DAY TO SECOND and ROWID decode into typed values.
11. Login failures map to real errors: REFUSE packets to `ORA-12514`-style messages, and a MARKER during
    auth phase two to `ORA-01017`.

Known limitation: LONG / LONG RAW columns in user queries are not decodable (garbled or underflow).
dblite reads LONG dictionary columns through inline `WITH FUNCTION ... RETURN CLOB` instead.

12. Query, define re-execute and fetch responses that span several TNS packets are reassembled
    (`Connection::parse_across_packets`, `ConnectionInner::append_next_packet`). Upstream parsed only the first
    packet, so a page whose rows outgrew one packet (LOB locators, wide columns) failed with
    "buffer underflow: need N bytes but only M available". The query and fetch parsers now report an underflow
    when the data ends before the terminal message (ERROR / STATUS / END_OF_RESPONSE), and the caller reads the
    next packet and parses again. Live tests: `lob_columns_next_to_dates_and_other_types`,
    `repeated_values_and_multiple_pages_with_lob_columns`, `responses_larger_than_one_packet_are_reassembled`.
