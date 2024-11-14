"use client";
import { useState, useCallback, useMemo } from "react";
import { message, Tag } from "antd";
import { format } from "date-fns";
import { Avatar } from "@radix-ui/react-avatar";
import Image from "next/image";
import axiosInstance from "@/utils/axiosInstance";
import { CameraCustomer, MappingFormValues, SapoCustomer } from "./types";
import React from "react";
import CustomerModal from "./CustomerModal";
import { imageDomains } from "@/utils/constants";
import defaultImage from "@/assets/images/default_image.jpg";

const CustomerCard: React.FC<{
  customer: CameraCustomer;
  setCustomers: React.Dispatch<React.SetStateAction<CameraCustomer[]>>;
  sapoCustomers: SapoCustomer[];
}> = React.memo(({ customer, setCustomers, sapoCustomers }) => {
  const [isButtonDisabled, setIsButtonDisabled] = useState(true);
  const [isModalOpen, setIsModalOpen] = useState(false);
  const [mappingLoading, setMappingLoading] = useState(false);

  const [customerProcessData, setCustomerProcessData] =
    useState<CameraCustomer>(customer);
  const [isVIP, setIsVip] = useState(
    customerProcessData.mappingStatus === "MAPPED",
  );

  const [mappingForm, setMappingForm] = useState<MappingFormValues>({
    mappingStatus: customerProcessData.mappingStatus,
    sourceCustomerId: customerProcessData.sourceCustomerId,
    notiEnabled: customerProcessData.notiEnabled,
    note: customerProcessData.note,
  });
  const toggleModal = useCallback(
    (isOpen: boolean) => {
      setIsModalOpen(isOpen);
      setMappingLoading(false);
      setIsButtonDisabled(true);
      setIsVip(customerProcessData.mappingStatus === "MAPPED");
      setMappingForm({
        mappingStatus: customerProcessData.mappingStatus,
        sourceCustomerId: customerProcessData.sourceCustomerId,
        notiEnabled: customerProcessData.notiEnabled,
        note: customerProcessData.note,
      });
    },
    [
      customerProcessData.mappingStatus,
      customerProcessData.note,
      customerProcessData.notiEnabled,
      customerProcessData.sourceCustomerId,
    ],
  );

  const callMappingApi = useCallback(async () => {
    const { id: CameraCustomerId } = customer;
    const requestBody: {
      mappingStatus: "MAPPED" | "UNMAP";
      sourceCustomerId: string;
      notiEnabled: boolean;
      note: string;
    } = {
      mappingStatus: mappingForm.mappingStatus,
      sourceCustomerId: mappingForm.sourceCustomerId,
      notiEnabled: mappingForm.notiEnabled,
      note: mappingForm.mappingStatus === "UNMAP" ? "" : mappingForm.note,
    };
    setMappingLoading(true);
    try {
      const response = await axiosInstance.patch(
        `/camera-customers/${CameraCustomerId}`,
        requestBody,
      );
      const newData: CameraCustomer = {
        ...customerProcessData,
        sourceCustomerId: mappingForm.sourceCustomerId,
        notiEnabled: mappingForm.notiEnabled,
        note: mappingForm.mappingStatus === "UNMAP" ? "" : mappingForm.note,
      };
      setCustomerProcessData(newData);
      setMappingLoading(false);
      message.success("Mapping successfully");
      return response.data;
    } catch {
      message.error("Mapping failed");
    } finally {
      setMappingLoading(false);
    }
  }, [customer, mappingForm, customerProcessData]);

  const handleNoteMessageChange = (
    event: React.ChangeEvent<HTMLTextAreaElement>,
  ) => {
    setMappingForm((prev) => ({
      ...prev,
      note: event.target.value,
    }));
    setIsButtonDisabled(false);
  };
  const handleNotificationChange = (checked: boolean) => {
    setMappingForm((prev) => ({
      ...prev,
      notiEnabled: checked,
    }));
    setIsButtonDisabled(false);
  };
  const handleSelectSapoData = (customerId: string) => {
    if (customerProcessData.mappingStatus === "UNMAP") {
      setMappingForm((prev) => ({
        ...prev,
        mappingStatus: "MAPPED",
      }));
    }
    setMappingForm((prev) => ({
      ...prev,
      sourceCustomerId: customerId || "",
    }));
    setIsButtonDisabled(false);
  };
  const handleChangeTagStatus = useCallback(
    (event: React.MouseEvent<HTMLElement>) => {
      if (customer.mappingStatus === "UNMAP") return;
      if (!isModalOpen) {
        event.stopPropagation();
      }
      if (customer.mappingStatus === "MAPPED" && isModalOpen) {
        event.preventDefault();
        setIsVip(!isVIP);
        setMappingForm((prev) => ({
          ...prev,
          mappingStatus: isVIP ? "UNMAP" : "MAPPED",
        }));
      }
      setIsButtonDisabled(false);
    },
    [customer.mappingStatus, isModalOpen, isVIP],
  );

  const handleOk = useCallback(async () => {
    await callMappingApi();
    if (customerProcessData.mappingStatus !== mappingForm.mappingStatus) {
      setIsVip(true);
      setCustomers((prev) =>
        prev.filter((cust) => cust.id !== customerProcessData.id),
      );
    }
    toggleModal(false);
  }, [
    callMappingApi,
    customerProcessData.id,
    customerProcessData.mappingStatus,
    mappingForm.mappingStatus,
    setCustomers,
    toggleModal,
  ]);

  const isValidUrl = (url: string) => {
    try {
      const { hostname } = new URL(url);
      return imageDomains.includes(hostname);
    } catch (error) {
      return false;
    }
  };
  const CustomerCardContent = useMemo(
    () => (
      <div className="flex items-center justify-between w-full space-x-4 px-2">
        <div className="flex items-center gap-4">
          <Avatar>
            <Image
              src={"https://img.pikbest.com/origin/09/19/03/61zpIkbEsTGjk.jpg!w700wp"
              }
              alt={`${customerProcessData.name}'s avatar`}
              className="w-10 h-10 rounded-full"
              width={100}
              height={100}
            />
          </Avatar>
          <div>
            <h3 className="font-semibold">{customerProcessData.name}</h3>
            <p className="text-sm text-gray-500">
              {format(
                new Date(customerProcessData.lastVisitTime),
                "dd-MM HH:mm",
              )}
            </p>
            <p className="text-sm text-gray-500">{customerProcessData.id}</p>
          </div>
        </div>
        <Tag
          className="cursor-pointer mr-6"
          onClick={handleChangeTagStatus}
          color={mappingForm.mappingStatus === "MAPPED" ? "green" : "red"}
        >
          {mappingForm.mappingStatus === "MAPPED"
            ? "VIP"
            : mappingForm.mappingStatus}
        </Tag>
      </div>
    ),
    [customer, isVIP, handleChangeTagStatus],
  );
  return (
    <>
      <div
        onClick={() => toggleModal(true)}
        className="flex items-center justify-between cursor-pointer transition-all duration-300 hover:bg-white hover:rounded-lg hover:shadow"
      >
        {CustomerCardContent}
      </div>
      <CustomerModal
        isModalOpen={isModalOpen}
        isButtonDisabled={isButtonDisabled}
        mappingLoading={mappingLoading}
        customerProcessData={customerProcessData}
        mappingForm={mappingForm}
        CustomerCardContent={CustomerCardContent}
        sapoCustomers={sapoCustomers}
        toggleModal={toggleModal}
        handleOk={handleOk}
        handleSelectSapoData={handleSelectSapoData}
        handleNotificationChange={handleNotificationChange}
        handleNoteMessageChange={handleNoteMessageChange}
      />
    </>
  );
});

CustomerCard.displayName = "CustomerCard";
export default CustomerCard;
