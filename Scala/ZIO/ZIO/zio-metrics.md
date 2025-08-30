object MetricJson {

  // Converts MetricValue to Any (Double or String)
  def metricToAny(m: MetricValue): Any = m match {
    case MetricValue.Gauge(d) => d
    case MetricValue.Text(s)  => s
  }

  // Recursively insert a value into a Map by dotted path
  def insert(path: List[String], value: Any, map: Map[String, Any]): Map[String, Any] =
    path match {
      case key :: Nil =>
        map.updated(key, value)
      case key :: rest =>
        val child = map.get(key) match {
          case Some(existing: Map[String, Any] @unchecked) => existing
          case _                                           => Map.empty[String, Any]
        }
        map.updated(key, insert(rest, value, child))
      case Nil => map
    }

  // Build the nested map from snapshot
  def buildNestedMap(snapshot: Map[String, MetricValue]): Map[String, Any] =
    snapshot.foldLeft(Map.empty[String, Any]) { case (acc, (k, v)) =>
      insert(k.split('.').toList, metricToAny(v), acc)
    }

  // zio-json encoder for Map[String, Any]
  implicit val anyEncoder: JsonEncoder[Any] = new JsonEncoder[Any] {
    def unsafeEncode(a: Any, indent: Option[Int], out: zio.json.internal.Write): Unit = a match {
      case d: Double    => JsonEncoder[Double].unsafeEncode(d, indent, out)
      case s: String    => JsonEncoder[String].unsafeEncode(s, indent, out)
      case m: Map[_, _] => JsonEncoder[Map[String, Any]].unsafeEncode(m.asInstanceOf[Map[String, Any]], indent, out)
      case _            => JsonEncoder[String].unsafeEncode(a.toString, indent, out)
    }
  }

  implicit val mapEncoder: JsonEncoder[Map[String, Any]] = JsonEncoder.keyValuePairs[String, Any]

  // Convert snapshot to JSON string
  def snapshotToJson(snapshot: Map[String, MetricValue]): String =
    buildNestedMap(snapshot).toJson
}
