Make the analyzer infer what preg_match_all() writes to its matches argument when the pattern is a literal string and the flags are omitted or a literal, valid combination of PREG_PATTERN_ORDER, PREG_SET_ORDER, PREG_OFFSET_CAPTURE and PREG_UNMATCHED_AS_NULL.

The type mirrors PHP at runtime: the whole match and every capturing group under its number and name, keys optional only where PHP can omit them, null for unmatched groups and each capture paired with its integer offset when the flags ask. The whole match, or a group taking part in every match, is a non-empty-string when it cannot match an empty string, and lists are non-empty where the call is known to have matched.

Other calls keep the declared type, as do patterns that do not compile or reuse a group name.
