# 7.1 topic 名规则

## topic

1. topic 名 不能为空字符串
2. topic 名 不能为 `.` 或者 `..`
3. topic 名 不能超过 249 个字符
4. topic 名 由 `[a-zA-Z0-9._-]` 这些字符组成

``` scala
public static final String LEGAL_CHARS = "[a-zA-Z0-9._-]";

private static final int MAX_NAME_LENGTH = 249;

public static void validate(String topic) {
    if (topic.isEmpty())
        throw new InvalidTopicException("Topic name is illegal, it can't be empty");
    if (topic.equals(".") || topic.equals(".."))
        throw new InvalidTopicException("Topic name cannot be \".\" or \"..\"");
    if (topic.length() > MAX_NAME_LENGTH)
        throw new InvalidTopicException("Topic name is illegal, it can't be longer than " + MAX_NAME_LENGTH +
                " characters, topic name: " + topic);
    if (!containsValidPattern(topic))
        throw new InvalidTopicException("Topic name \"" + topic + "\" is illegal, it contains a character other than " +
                "ASCII alphanumerics, '.', '_' and '-'");
}

/**
 * Valid characters for Kafka topics are the ASCII alphanumerics, '.', '_', and '-'
 */
static boolean containsValidPattern(String topic) {
    for (int i = 0; i < topic.length(); ++i) {
        char c = topic.charAt(i);

        // We don't use Character.isLetterOrDigit(c) because it's slower
        boolean validChar = (c >= 'a' && c <= 'z') || (c >= '0' && c <= '9') || (c >= 'A' && c <= 'Z') || c == '.' ||
                c == '_' || c == '-';
        if (!validChar)
            return false;
    }
    return true;
}
```

## group id

group id 没有找到限制

``` scala
private def isValidGroupId(groupId: String, api: ApiKeys): Boolean = {
  api match {
    case ApiKeys.OFFSET_COMMIT | ApiKeys.OFFSET_FETCH | ApiKeys.DESCRIBE_GROUPS | ApiKeys.DELETE_GROUPS =>
      // For backwards compatibility, we support the offset commit APIs for the empty groupId, and also
      // in DescribeGroups and DeleteGroups so that users can view and delete state of all groups.
      groupId != null
    case _ =>
      // The remaining APIs are groups using Kafka for group coordination and must have a non-empty groupId
      groupId != null && !groupId.isEmpty
  }
}
```
