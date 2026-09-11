(ns laundrycoord.advisor
  "Laundry Plant Scheduling Coordination Advisor — proposing a plant
  scheduling/logistics coordination operation (log a work record,
  schedule a crew operation, flag a safety concern, coordinate a
  detergent/linen-stock supply order) from a crew roster, plant
  registration and safety-reporting policy. Swappable mock/llm; the
  advisor ONLY proposes — `laundrycoord.governor` independently gates
  every proposal and always escalates safety concerns and
  above-threshold supply orders. The advisor never proposes to
  directly finalize a machine-operation-execution decision (e.g.
  deciding to proceed with a specific washing, drying or pressing
  run), or a plant-safety-clearance decision (e.g. declaring a washer,
  dryer or press safe to operate, or a laundry batch safe for
  shipment), and never proposes to override a plant safety officer's
  judgment — those stay permanently out of this actor's scope.
  Modeled closely on cloud-itonami-isco-8152's millcoord.advisor.

  A proposal: {:op :log-work-record|:schedule-crew-operation|
               :flag-safety-concern|:coordinate-supply-order
               :effect :propose :launderer-id str :plant-id str
               :cost number :hazard-type kw :task str :stake kw
               :confidence n :rationale str}")

(defprotocol Advisor
  (-advise [advisor store request] "request -> proposal map"))

(defn- rationale-for [op launderer-id plant-id hazard-type]
  (case op
    :log-work-record
    (str "logged work record for launderer " launderer-id " at plant " plant-id)

    :schedule-crew-operation
    (str "scheduled crew operation for washing/drying/pressing task at plant " plant-id)

    :flag-safety-concern
    (str "flagged " (name (or hazard-type :hazard)) " concern for launderer "
         launderer-id " at plant " plant-id " — routed for plant safety officer review")

    :coordinate-supply-order
    (str "coordinated detergent/linen-stock supply order for launderer " launderer-id " at plant " plant-id)

    (str "proposed " (name op) " for launderer " launderer-id " at plant " plant-id)))

(defn- infer [_store {:keys [op stake launderer-id plant-id cost hazard-type task]
                       :as request}]
  {:op op
   :effect :propose
   :launderer-id launderer-id
   :plant-id plant-id
   :cost cost
   :hazard-type hazard-type
   :task task
   :stake (or stake :low)
   :confidence (case (or stake :low) :high 0.7 :medium 0.85 :low 0.95)
   :rationale (rationale-for op launderer-id plant-id hazard-type)})

(defn mock-advisor []
  (reify Advisor
    (-advise [_ store request] (infer store request))))

(def ^:private system-prompt
  "You are a laundry machine operators plant scheduling/logistics
   coordination advisor. Given a request, propose an :op (one of
   :log-work-record, :schedule-crew-operation, :flag-safety-concern,
   :coordinate-supply-order), the :launderer-id, :plant-id, and any
   :cost/:hazard-type/:task fields, an honest :confidence and a
   :stake. Never propose an op outside this closed list, and never
   propose to directly finalize a machine-operation-execution decision
   (e.g. deciding to proceed with a specific washing, drying or
   pressing run), or a plant-safety-clearance decision (e.g. declaring
   a washer, dryer or press safe to operate, or a laundry batch safe
   for shipment), or to override a plant safety officer's judgment —
   those are always out of this actor's scope; it coordinates plant
   scheduling/logistics only and never operates laundry-machine
   equipment or clears machines/batches for operation/shipment itself.
   Heat/steam-burn hazard, entanglement-hazard (rotating drums and
   pressing mechanisms) and detergent/chemical-exposure safety
   concerns always require human sign-off regardless of confidence.")

(defn- parse-proposal [content]
  (try
    (let [p (read-string content)]
      (if (map? p)
        (assoc p :effect :propose)
        {:op :unknown :effect :propose :confidence 0.0 :stake :high
         :rationale "unparseable LLM response"}))
    (catch #?(:clj Exception :cljs js/Error) _
      {:op :unknown :effect :propose :confidence 0.0 :stake :high
       :rationale "LLM response parse failure"})))

(defn llm-advisor
  [chat-model model-generate-fn gen-opts]
  (reify Advisor
    (-advise [_ _store request]
      (let [msgs [{:role :system :content system-prompt}
                  {:role :user :content (str "operation request: " (pr-str request))}]
            resp (model-generate-fn chat-model msgs gen-opts)]
        (parse-proposal (:content resp))))))
