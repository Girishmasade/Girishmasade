// Components/Address.jsx
import { useCallback, useEffect, useMemo, useRef, useState } from "react";
import Checkbox from "../ReusableComponents/Checkbox";
import AdAddressForm from "./AddAddressForm";
import {
  useCreateAddressMutation,
  useUpdateAddressMutation,
  useDeleteAddressMutation,
} from "../redux/apis/addressApi";
import AddressCard from "../ReusableComponents/AddressCard";
import toast from "react-hot-toast";
import { useProfileQuery } from "../redux/apis/scmAuthApi";
import { decryptApiResponse } from "../utils/CryptoStorage";
import { useSelector } from "react-redux";

export default function AddressStep({
  setSelectedDeliveryAddress,
  setSelectedBillingAddress,
  setIsAddressFormOpen,
  setGSTInvoiceEnabled,
  isAddressFormOpen,
}) {
  const hasLocalAddressChangesRef = useRef(false);

  const setDeliveryRef = useRef(setSelectedDeliveryAddress);
  const setBillingRef = useRef(setSelectedBillingAddress);
  const setFormOpenRef = useRef(setIsAddressFormOpen);
  const setGSTRef = useRef(setGSTInvoiceEnabled);

  setDeliveryRef.current = setSelectedDeliveryAddress;
  setBillingRef.current = setSelectedBillingAddress;
  setFormOpenRef.current = setIsAddressFormOpen;
  setGSTRef.current = setGSTInvoiceEnabled;

  const stableSetDelivery = useCallback(
    (addr) => setDeliveryRef.current(addr),
    [],
  );
  const stableSetBilling = useCallback(
    (addr) => setBillingRef.current(addr),
    [],
  );
  const stableSetFormOpen = useCallback(
    (val) => setFormOpenRef.current(val),
    [],
  );
  const stableSetGST = useCallback((val) => setGSTRef.current(val), []);

  const preserveScroll = (fn) => {
    const y = window.scrollY;
    fn();
    requestAnimationFrame(() => {
      window.scrollTo(0, y);
    });
  };

  const isLoggedIn = useSelector((state) => !!state.auth.token);
  const {
    data: profileApiData,
    isLoading,
    isError,
  } = useProfileQuery(undefined, { skip: !isLoggedIn });

  const profileAddresses = useMemo(() => {
    const raw = profileApiData?.data;
    let resolved = null;
    if (raw) {
      resolved = typeof raw === "string" ? decryptApiResponse(raw) : raw;
    } else {
      const stored = localStorage.getItem("userInfo");
      resolved = stored ? decryptApiResponse(stored) : null;
    }
    return resolved?.addresses || [];
  }, [profileApiData]);

  const [addressesState, setAddressesState] = useState([]);
  const addresses = addressesState;

  const [createAddress] = useCreateAddressMutation();
  const [updateAddress] = useUpdateAddressMutation();
  const [deleteAddress] = useDeleteAddressMutation();

  const [selectedDeliveryId, setSelectedDeliveryId] = useState(null);
  const [selectedBillingId, setSelectedBillingId] = useState(null);

  const [gstEnabled, setGstEnabled] = useState(false);
  const [sameAsDelivery, setSameAsDelivery] = useState(false);

  const [showForm, setShowForm] = useState(false);
  const [formMode, setFormMode] = useState("create");
  const [addressType, setAddressType] = useState("delivery");
  const [editingAddress, setEditingAddress] = useState(null);
  const [showAllDelivery, setShowAllDelivery] = useState(false);
  const [showAllBilling, setShowAllBilling] = useState(false);

  useEffect(() => {
    if (!isAddressFormOpen && showForm) {
      setShowForm(false);
      setEditingAddress(null);
    }
  }, [isAddressFormOpen, showForm]);

  useEffect(() => {
    if (hasLocalAddressChangesRef.current) return;
    setAddressesState((prev) => {
      if (prev.length > 0 && prev.length === profileAddresses.length) {
        const unchanged = prev.every((addr, i) => {
          const p = profileAddresses[i];
          return (
            p &&
            p._id === addr._id &&
            (p.updatedAt || p.createdAt || "") ===
              (addr.updatedAt || addr.createdAt || "")
          );
        });
        if (unchanged) return prev;
      }
      return Array.isArray(profileAddresses) ? profileAddresses : [];
    });
  }, [profileAddresses]);

  const deliveryAddresses = useMemo(
    () => addresses.filter((a) => a?.addressTypeHint === "DELIVERY"),
    [addresses],
  );

  const billingaddress = useMemo(() => {
    if (gstEnabled) {
      return addresses.filter(
        (a) => a?.addressTypeHint === "BILLING" && Boolean(a?.gstNumber),
      );
    }
    return addresses.filter(
      (a) => a?.addressTypeHint === "BILLING" && !a?.gstNumber,
    );
  }, [addresses, gstEnabled]);

  const lastDeliveryRef = useRef(null);
  const lastBillingRef = useRef(null);

  const resetBilling = useCallback(() => {
    setSelectedBillingId(null);
    stableSetBilling(null);
    lastBillingRef.current = null;
  }, [stableSetBilling]);

  const resetDeliveryAfterDelete = useCallback(
    (deletedId, sourceAddresses) => {
      if (selectedDeliveryId !== deletedId) return;
      const remaining = sourceAddresses.filter(
        (a) => a.addressTypeHint === "DELIVERY" && a._id !== deletedId,
      );
      if (remaining.length > 0) {
        setSelectedDeliveryId(remaining[0]._id);
        stableSetDelivery(remaining[0]);
      } else {
        setSelectedDeliveryId(null);
        stableSetDelivery(null);
      }
    },
    [selectedDeliveryId, stableSetDelivery],
  );

  const resetBillingAfterDelete = useCallback(
    (deletedId, sourceAddresses) => {
      if (selectedBillingId !== deletedId) return;
      const remaining = sourceAddresses.filter((a) => {
        const isBillingType = a.addressTypeHint === "BILLING";
        const gstMatch = gstEnabled ? !!a.gstNumber : !a.gstNumber;
        return isBillingType && gstMatch && a._id !== deletedId;
      });
      if (remaining.length > 0) {
        setSelectedBillingId(remaining[0]._id);
        stableSetBilling(remaining[0]);
      } else {
        resetBilling();
      }
    },
    [selectedBillingId, gstEnabled, stableSetBilling, resetBilling],
  );

  useEffect(() => {
    if (deliveryAddresses?.length && !selectedDeliveryId) {
      const def =
        deliveryAddresses.find((a) => a.isDefaultDelivery) ||
        deliveryAddresses[0];
      setSelectedDeliveryId(def._id);
      lastDeliveryRef.current = def._id;
      stableSetDelivery(def);
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [deliveryAddresses]);

  useEffect(() => {
    if (!deliveryAddresses?.length) {
      setSelectedDeliveryId(null);
      stableSetDelivery(null);
      lastDeliveryRef.current = null;
    }
    if (!billingaddress?.length) {
      resetBilling();
    }
  }, [deliveryAddresses, billingaddress, stableSetDelivery, resetBilling]);

  useEffect(() => {
    if (!selectedDeliveryId || !deliveryAddresses?.length) return;
    const del = deliveryAddresses.find((a) => a._id === selectedDeliveryId);
    if (del && lastDeliveryRef.current !== del._id) {
      lastDeliveryRef.current = del._id;
      stableSetDelivery(del);
    }
  }, [selectedDeliveryId, deliveryAddresses, stableSetDelivery]);

  useEffect(() => {
    if (!selectedBillingId) return;
    if (sameAsDelivery) {
      const del = deliveryAddresses.find((a) => a._id === selectedBillingId);
      if (del && lastBillingRef.current !== del._id) {
        lastBillingRef.current = del._id;
        stableSetBilling(del);
      }
      return;
    }
    const bill = billingaddress.find((a) => a._id === selectedBillingId);
    if (bill && lastBillingRef.current !== bill._id) {
      lastBillingRef.current = bill._id;
      stableSetBilling(bill);
    }
  }, [
    selectedBillingId,
    billingaddress,
    deliveryAddresses,
    sameAsDelivery,
    stableSetBilling,
  ]);

  useEffect(() => {
    if (gstEnabled) {
      setSameAsDelivery(false);
      resetBilling();
    } else {
      resetBilling();
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [gstEnabled]);

  useEffect(() => {
    stableSetGST(gstEnabled);
  }, [gstEnabled, stableSetGST]);

  useEffect(() => {
    if (gstEnabled || !sameAsDelivery) return;
    const delivery = deliveryAddresses.find(
      (a) => a._id === selectedDeliveryId,
    );
    if (delivery) {
      setSelectedBillingId(delivery._id);
      lastBillingRef.current = delivery._id;
      stableSetBilling(delivery);
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [sameAsDelivery, selectedDeliveryId, deliveryAddresses]);

  const handleCloseForm = useCallback(() => {
    setShowForm(false);
    setEditingAddress(null);
    stableSetFormOpen(false);
    requestAnimationFrame(() => {
      window.scrollTo({ top: 0, behavior: "smooth" });
    });
  }, [stableSetFormOpen]);

  if (showForm) {
    return (
      <AdAddressForm
        mode={formMode}
        type={addressType}
        initialData={editingAddress}
        sameAsDelivery={sameAsDelivery}
        selectedDeliveryId={selectedDeliveryId}
        gstEnabled={gstEnabled}
        setGstEnabled={setGstEnabled}
        savedGst={addresses
          .filter((add) => add?.gstNumber && add?.addressTypeHint === "BILLING")
          .map((data) => data?.gstNumber)}
        onCancel={handleCloseForm}
        onBack={handleCloseForm}
        onSave={async (payload) => {
          try {
            if (formMode === "create") {
              const res = await createAddress(payload).unwrap();
              const createdAddress =
                res?.data?.address || res?.data || res?.address || null;
              toast.success("Address added successfully");
              if (createdAddress?._id) {
                hasLocalAddressChangesRef.current = true;
                setAddressesState((prev) => [createdAddress, ...prev]);
              }
              if (addressType === "delivery" && createdAddress?._id) {
                setSelectedDeliveryId(createdAddress._id);
                stableSetDelivery(createdAddress);
                lastDeliveryRef.current = createdAddress._id;
              } else {
                if (createdAddress?._id) {
                  setSelectedBillingId(createdAddress._id);
                  stableSetBilling(createdAddress);
                  lastBillingRef.current = createdAddress._id;
                }
              }
            } else {
              const updateRes = await updateAddress({
                id: editingAddress._id,
                payload,
              }).unwrap();
              const updatedFromApi =
                updateRes?.data?.address ||
                updateRes?.data ||
                updateRes?.address ||
                null;
              const updatedAddress = updatedFromApi?._id
                ? updatedFromApi
                : { ...editingAddress, ...payload };

              hasLocalAddressChangesRef.current = true;
              setAddressesState((prev) =>
                prev.map((a) =>
                  a._id === editingAddress._id
                    ? { ...a, ...updatedAddress }
                    : a,
                ),
              );
              if (selectedDeliveryId === editingAddress._id) {
                stableSetDelivery({ ...editingAddress, ...updatedAddress });
              }
              if (selectedBillingId === editingAddress._id) {
                stableSetBilling({ ...editingAddress, ...updatedAddress });
              }
              toast.success("Address updated successfully");
            }
            handleCloseForm();
          } catch (err) {
            console.error("Address save failed:", err);
            toast.error(
              err?.data?.message || "Failed to save address. Please try again.",
            );
          }
        }}
      />
    );
  }

  if (isLoading) {
    return <div className="p-3">Loading addresses...</div>;
  }

  if (isError) {
    return <div className="p-3 text-red-500">Failed to load addresses</div>;
  }

  return (
    <div className="flex flex-col">
      {/* ── GST Banner — always visible, never scrolls ── */}
      <div className="bg-[#3f5fa8] text-white rounded-md p-4 flex gap-3 flex-shrink-0">
        <Checkbox
          checked={gstEnabled}
          variant="inverse"
          onChange={(checked) => {
            preserveScroll(() => {
              setGstEnabled(checked);
              localStorage.setItem("GstEnable", "true");
              if (checked) setSameAsDelivery(false);
            });
          }}
        />
        <div>
          <p className="font-semibold">Get GST Invoice</p>
          <p className="text-sm opacity-90">
            Claim input tax credit on your purchase
          </p>
        </div>
      </div>

      {/* ── Scrollable address area ── */}
      <div className="mt-3 space-y-3 overflow-y-auto max-h-[38vh] pr-1">

        {/* ── Delivery Address ── */}
        <h3 className="font-semibold text-sm mb-1">Delivery Address</h3>
        <div className="space-y-2">
          {deliveryAddresses
            .filter(
              (addr) => showAllDelivery || addr._id === selectedDeliveryId,
            )
            .map((addr) => (
              <AddressCard
                key={addr._id}
                address={addr}
                selected={selectedDeliveryId === addr._id}
                onSelect={() => {
                  setSelectedDeliveryId(addr._id);
                  stableSetDelivery(addr);
                  setShowAllDelivery(false);
                }}
                onEdit={() => {
                  setFormMode("edit");
                  setAddressType("delivery");
                  setEditingAddress(addr);
                  setShowForm(true);
                  stableSetFormOpen(true);
                }}
                onDelete={async () => {
                  try {
                    await deleteAddress(addr._id).unwrap();
                    hasLocalAddressChangesRef.current = true;
                    const nextAddresses = addresses.filter(
                      (a) => a._id !== addr._id,
                    );
                    setAddressesState(nextAddresses);
                    toast.success("Address deleted successfully");
                    resetDeliveryAfterDelete(addr._id, nextAddresses);
                  } catch (err) {
                    console.error("Delete failed", err);
                    toast.error("Failed to delete address");
                  }
                }}
              />
            ))}

          {deliveryAddresses.length > 1 && (
            <button
              onClick={() => setShowAllDelivery((prev) => !prev)}
              className="text-[#3B5AA1] text-xs font-medium"
            >
              {showAllDelivery ? "Hide Addresses" : "Change Address"}
            </button>
          )}

          <div
            className="border-dashed border p-2 cursor-pointer text-[#3B5AA1] text-sm rounded"
            onClick={() => {
              setFormMode("create");
              setAddressType("delivery");
              setEditingAddress(null);
              setShowForm(true);
              stableSetFormOpen(true);
            }}
          >
            + Add New Address
          </div>
        </div>

        {/* ── Billing Address — always rendered, never hidden ── */}
        <div>
          {/* Header: "Billing Address" + inline "Same as delivery" checkbox */}
          <div className="flex items-center gap-2 mb-2">
            <h3 className="font-semibold text-sm">Billing Address</h3>
            {!gstEnabled && (
              <label className="flex items-center gap-1.5 cursor-pointer select-none">
                <Checkbox
                  checked={sameAsDelivery}
                  onChange={(checked) => {
                    preserveScroll(() => {
                      setSameAsDelivery(checked);
                      if (!checked) resetBilling();
                    });
                  }}
                />
                <span className="text-xs text-gray-500">Same as delivery</span>
              </label>
            )}
          </div>

          {/* 
            When sameAsDelivery is ON  → show the selected delivery address as a
            read-only billing card (visible and scrollable).
            When sameAsDelivery is OFF → show the normal billing address picker.
          */}
          {!gstEnabled && sameAsDelivery ? (
            /* ── Mirror of selected delivery address ── */
            <div className="space-y-1">
              {deliveryAddresses
                .filter((a) => a._id === selectedDeliveryId)
                .map((addr) => (
                  <AddressCard
                    key={addr._id}
                    address={{
                      ...addr,
                      name: addr.organizationName || addr.name,
                    }}
                    selected
                    hideActions
                  />
                ))}
            </div>
          ) : (
            /* ── Full billing address picker ── */
            <div className="space-y-2">
              {billingaddress
                .filter(
                  (addr) =>
                    showAllBilling ||
                    !selectedBillingId ||
                    addr._id === selectedBillingId,
                )
                .map((addr) => (
                  <AddressCard
                    key={addr._id}
                    address={{
                      ...addr,
                      name: addr.organizationName,
                    }}
                    selected={selectedBillingId === addr._id}
                    onSelect={() => {
                      setSelectedBillingId(addr._id);
                      stableSetBilling(addr);
                      setShowAllBilling(false);
                    }}
                    onEdit={() => {
                      setFormMode("edit");
                      setAddressType("billing");
                      setEditingAddress(addr);
                      setShowForm(true);
                      stableSetFormOpen(true);
                    }}
                    onDelete={async () => {
                      try {
                        await deleteAddress(addr._id).unwrap();
                        hasLocalAddressChangesRef.current = true;
                        const nextAddresses = addresses.filter(
                          (a) => a._id !== addr._id,
                        );
                        setAddressesState(nextAddresses);
                        toast.success("Billing address deleted");
                        resetBillingAfterDelete(addr._id, nextAddresses);
                      } catch (err) {
                        console.error("Delete failed", err);
                        toast.error("Failed to delete billing address");
                      }
                    }}
                  />
                ))}

              {billingaddress.length > 1 && (
                <button
                  onClick={() => setShowAllBilling((prev) => !prev)}
                  className="text-[#3B5AA1] text-xs font-medium"
                >
                  {showAllBilling ? "Hide Addresses" : "Change Address"}
                </button>
              )}

              <div
                className="border-dashed border p-2 cursor-pointer text-[#3B5AA1] text-sm rounded"
                onClick={() => {
                  setFormMode("create");
                  setAddressType("billing");
                  setEditingAddress(null);
                  setShowForm(true);
                  stableSetFormOpen(true);
                }}
              >
                + Add Billing Address
              </div>
            </div>
          )}
        </div>

      </div>
    </div>
  );
}
