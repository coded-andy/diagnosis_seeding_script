alias Onemedical.Indexing.Diagnosis.{
  SyncDiagnosisLibrary,
  DeleteDuplicates,
  CombineStandards
}

require Logger
max_page = System.get_env("OCL_MAX_PAGE") || "0"
System.put_env("OCL_MAX_PAGE", "51")

tenants = [Onemedical.Repo]



Logger.info(
  message: "Running Elastic Search data migration",
  tenants: tenants
)


  ocl_task = SyncDiagnosisLibrary.fetch_from_ocl_task(tenants)

  sync_task = Task.async(fn ->
    Logger.metadata(stage: :sync)
    SyncDiagnosisLibrary.sync_missing_records(tenants)
  end)



case Task.await_many([sync_task, ocl_task], :infinity) do
  [_sync_result, ocl_result] -> 
    SyncDiagnosisLibrary.upload_from_ocl(ocl_result)
  other ->
    Logger.error("Unexpected result: #{inspect(other)}")
    {:error, :unexpected_result}
end

Logger.info(
  message: "Finished syncing all records, starting duplicates cleanup.",
  tenants: tenants
)

tenants
|> DeleteDuplicates.find_and_delete_all_duplicates()

Logger.info(
  message:
    "Finished duplicates clean up, starting standards combination.",
  tenants: tenants
)

tenants
|> CombineStandards.combine_record_standards()

Logger.info(
  message:
    "Finished combining standards, Diagnosis data sync completed!",
  tenants: tenants
)
System.put_env("OCL_MAX_PAGE", max_page)
